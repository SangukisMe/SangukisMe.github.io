---
title: "임베디드 OS 개발 프로젝트 10장 :: CONTEXT SWITCHING(1)"
excerpt: "컨텍스트 백업 및 복구하기"

toc: true # Table of Contents
toc_sticky: true # TOC를 고정해주는 역할 
# toc_label: # TOC의 제목 설정

categories:
  - Embedded RTOS
tags:
  - Embedded
  - RTOS
last_modified_at: 2021-07-29
---

<br/>

# 10장 tree

.  
├── Makefile  
├── boot  
│   ├── Entry.S  
│   ├── Handler.c  
│   └── Main.c  
├── hal  
│   ├── HalInterrupt.h  
│   ├── HalTimer.h  
│   ├── HalUart.h  
│   └── rvpb  
│       ├── Interrupt.c  
│       ├── Interrupt.h  
│       ├── Regs.c  
│       ├── Timer.c  
│       ├── Timer.h  
│       ├── Uart.c  
│       └── Uart.h  
├── include  
│   ├── ARMv7AR.h  
│   ├── MemoryMap.h  
│   ├── memio.h  
│   ├── stdarg.h  
│   ├── stdbool.h  
│   └── stdint.h  
├── kernel  
│   ├── **Kernel.c**  
│   ├── **Kernel.h**  
│   ├── task.c  
│   └── task.h  
├── lib  
│   ├── armcpu.c  
│   ├── armcpu.h  
│   ├── stdio.c  
│   ├── stdio.h  
│   ├── stdlib.c  
│   └── stdlib.h  
└── navilos.ld

<br/>

# 10.0 개요

- 태스크 : 동작하는 프로그램.

- 컨텍스트 : 동작하는 프로그램(태스크)의 정보

- 스케줄러 : 지금 실행 중인 태스크 다음에 실행할 태스크를 골라주는 녀석

- 컨텍스트 스위칭 : 컨텍스트를 어딘가에 저장하고 또 다른 어딘가에서 컨텍스트를 가져다가 프로세서 코어에 복구하여 다른 프로그램이 동작하게 하는 것.

- 나빌로스의 컨텍스트 스위칭 
  1. 현재 동작하고 있는 태스크의 컨텍스트를 현재 스택에 백업.
  2. 다음에 동작할 태스크 컨트롤 블록을 스케줄러를 통해 받기.
  3. 2에서 받은 태스크 컨트롤 블록에서 스택 포인터를 읽기.
  4. 3에서 읽은 태스크의 스택에서 컨텍스트를 읽어서 ARM 코어에 복구.
  5. 다음에 동작할 태스크의 직전 프로그램 실행 위치(pc)로 이동. 이 순간 해당 태스크가 동작하는 태스크가 된다. 


```bash
vi task.c
```

```c
static KernelTcb_t* sCurrent_tcb;
static KernelTcb_t* sNext_tcb;

/… 중략 …/

void Kernel_task_scheduler(void)
{
    sCurrent_tcb = &sTask_list[sCurrent_tcb_index];
    sNext_tcb = Scheduler_round_robin_algorithm();

    disable_irq();
    Kernel_task_context_switching();
    enable_irq();
}
```

- 컨텍스트 스위칭을 위한 스케줄러 함수 구현. 위에서 언급한 1~5번의 절차를 코드로 옮긴 것.
- sCurrent_tcb : 현재 동작 중인 태스크 컨트롤 블록의 포인터
- sNext_tcb : 라운드 로빈 알고리즘이 선택한 다음에 동작할 태스크 컨트롤 블록의 포인터
- 위의 두 포인터를 확보해 놓고 컨텍스트 스위칭 수행.

>스위칭을 할 때 인터럽트를 disable 해준다. 이는 task(process)와 interrupt 서비스 루틴 사이에서 공유 resources에 대한 Race Condition을 방지하기 위함이다.

```c
// task.c 

__attribute__ ((naked)) void Kernel_task_context_switching(void)
{
    __asm__ ("B Save_context");
    __asm__ ("B Restore_context");
}
```

- 컨텍스트 스위칭 함수. 실질적으로 스위칭을 일으키는 함수를 호출하는 함수이다.
- Save_context : 현재 동작하고 있는 태스크의 컨텍스트를 현재 스택에 백업. 호출 시 ARM 인스트럭션 B를 사용하여 LR을 변경하지 않는다.
- Restore_context : 다음 태스크의 스택에서 컨텍스트를 읽어서 ARM 코어에 복구. 호출 시 ARM 인스트럭션 B를 사용하여 LR을 변경하지 않는다.

>```__attribute__ ((naked))``` : GCC의 컴파일러 어트리뷰트 기능으로, 컴파일러가 함수를 컴파일할 때 자동으로 만드는 스택 백업, 복구, 리턴 관련 어셈블리어가 전혀 생성되지 않고 내부에 코딩한 코드 자체만 남기는 기능을 한다. 나빌로스는 컨텍스트를 스택에 백업하고 스택에서 복구할 것이므로 컨텍스트 스위칭을 할 때 되도록 스택을 그대로 유지하는 것이 좋다.
>
>일반적인 C 언어에서 함수 호출 시 스택 확보 및 LR에 리턴 주소를 넣어주어야 한다. 이러한 작업을 안하는것이 아닌 Save_context 와 Resotre_context 함수에서 어셈블리어를 이용해 직접 제어를 하여 문제 없이 함수를 호출하는 것처럼 설계를 할 것이다.

<br/>

# 10.1 컨텍스트 백업하기

```bash
vi task.c
```

```c
static __attribute__ ((naked)) void Save_context(void)
{
    // save current task context into the current task stack
    __asm__ ("PUSH {lr}");
    __asm__ ("PUSH {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12}");
    __asm__ ("MRS   r0, cpsr");
    __asm__ ("PUSH {r0}");
    // save current task stack pointer into the current TCB
    __asm__ ("LDR   r0, =sCurrent_tcb");
    __asm__ ("LDR   r0, [r0]");
    __asm__ ("STMIA r0!, {sp}");
}
```

- Save_context() 함수 구현. 현재 테스크의 컨텍스트를 스택에 백업한다.
  - LR을 스택에 푸시(PUSH) : KernelTaskContext_t의 pc 멤버 변수에 LR이 저장된다. 나중에 태스크가 다시 스케쥴링을 받았을 때 복귀하는 위치는 pc 멤버 변수가 저장하고 있으며, 이 위치는 Kernel_task_context_switching() 함수의 리턴 주소가 된다.
  - 범용 레지스터인 R0부터 R12까지를 스택에 푸시(PUSH) : ```__attribute__ ((naked))``` 컴파일러 어트리뷰트 지시어를 사용했으므로 스위칭 함수를 호출하기 직전의 값이 계속 유지되고 있다.
  - CPSR을 스택에 푸시(PUSH) : KernelTaskContext_t의 spsr 멤버 변수에 CPSR이 저장된다. 프로그램 상태 레지스터는 직접 메모리에 저장할 수 없으므로 R0를 사용한다.
  - 현재 동작 중인 태스크 컨텍스트 블록의 포인터 변수를 읽어와서, 포인터에 저장된 값을 읽는다. 포인터에 저장된 값이 주솟값이므로 r0로 온전한 메모리 위치를 읽은 뒤, 베이스 메모리 주소로 사용하기 위해 TCB의 SP에 저장한다.


>ARM 코어의 sp 레지스터가 아닌 태스크 컨트롤 블록 구조체의 첫번째 멤버 변수인 sp에 저장하는 것이다. 

>```c
>typedef struct KernelTaskContext_t
>{
>    uint32_t spsr;
>    uint32_t r0_r12[13];
>    uint32_t pc;
>} KernelTaskContext_t;
>```
>
>- spsr, r0_r12, pc 순서를 맞춰서 스택에 백업한다.
>- 스택은 메모리 주소가 큰 값에서 작은 값으로 진행된다. 따라서 pc, r0_r12, spsr 순서로 백업을 해야 의도한 자료 구조 의미에 맞는 메모리 주소에 값이 저장된다.

<br/>

# 10.2 컨텍스트 복구하기

```bash
vi task.c
```

```c
static __attribute__ ((naked)) void Restore_context(void)
{
    // restore next task stack pointer from the next TCB
    __asm__ ("LDR   r0, =sNext_tcb");
    __asm__ ("LDR   r0, [r0]");
    __asm__ ("LDMIA r0!, {sp}");
    // restore next task context from the next task stack
    __asm__ ("POP  {r0}");
    __asm__ ("MSR   cpsr, r0");
    __asm__ ("POP  {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12}");
    __asm__ ("POP  {pc}");
}
```

- Restore_context() 함수 구현. 다음 태스크의 컨텍스트를 ARM 코어에 복구한다.

- 형태는 Save_context() 함수와 비슷하며, 동작만 역순으로 진행 될 뿐이다.
  - sNext_tcb에서 스택 포인터 값을 읽어서 ARM 코어의 SP에 값을 쓴다.
  - 스택에 저장되어 있는 cpsr의 값을 꺼내서 ARM 코어의 CPSR에 팝(POP).
  - 스택에 저장되어 있는 R0부터 R12까지의 값을 ARM 코어의 범용 레지스터에 팝(POP). 이 시점 이후 R0 ~ R12까지의 레지스터 값을 변경하면 컨텍스크 복구에 실패하게 된다. 그래서 다른 작업을 하지 않고 바로 다음 스택 값인 pc를 ARM 코어의 PC에 팝(POP)하면서 태스크 코드로 점프한다.


>마지막 스택의 pc값이 POP 되는 순간 ARM 코어는 해당 태스크의 컨텍스트가 백업되기 직전의 코드 위치로 PC를 옮기고 실행을 이어한다. 이런 식으로 태스크 입장에서는 누락되는 코드 없이 그대로 이어서 프로그램이 계속 실행되는 것이다.

