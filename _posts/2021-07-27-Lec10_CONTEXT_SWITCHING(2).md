---
title: "임베디드 OS 개발 프로젝트 10장 :: CONTEXT SWITCHING(2)"
excerpt: "yield 만들기 및 커널 시작하기"

toc: true # Table of Contents
toc_sticky: true # TOC를 고정해주는 역할 
# toc_label: # TOC의 제목 설정

categories:
  - Embedded RTOS
tags:
  - Embedded
  - RTOS
last_modified_at: 2021-07-27
---

<br/>

# 10.3 yield 만들기

- 스케줄링(scheduling) : 스케줄러와 컨텍스트 스위칭을 합쳐서 스케줄링 이라고 한다.
- 시분할 시스템 : 각 태스크가 일정한 시간만 동작하고 다음 태스크로 전환되는 시스템.
- 선점형 멀티태스킹 시스템 : 태스크가 명시적으로 스케줄링을 요청하지 않았는데 커널이 강제로 스케줄링하는 시스템.
- 비선점형 멀티태스킹 시스템 : 태스크가 명시적으로 스케줄링을 요청하지 않으면 커널이 스케줄링하지 않는 시스템.
- 나빌로스는 시분할이 아닌 시스템에 비선점형 스케줄링을 사용한다. 이 말은 스케줄링하려면 테스크가 명시적으로 커널에 스케줄링을 요청해야 한다는 말이다. 태스크가 커널에 스케줄링을 요청하는 동작은 태스크가 CPU 자원을 다음 태스크에 양보한다는 의미로 해석할 수 있다. 일반적으로 **이런 동작을 하는 함수의 이름은 양보한다는 의미로 yield를 많이 사용한다.**

```bash
vi Kernel.h
```

```c
#ifndef KERNEL_KERNEL_H_
#define KERNEL_KERNEL_H_

#include "task.h"

void Kernel_yield(void);

#endif /* KERNEL_KERNEL_H_ */
```

- Kernel_yield() 함수 정의. 
- 호출만 하면 알아서 동작 하도록 만들 것이므로 별도의 파라미터나 리턴 값은 없다.

```bash
vi Kernel.c
```

```c
#include "stdint.h"
#include "stdbool.h"

#include "Kernel.h"

void Kernel_yield(void)
{
	Kernel_task_scheduler();
}
```

- Kernel_yield() 함수 구현.
- 그냥 Kernel_task_scheduler() 함수를 직접 호출하는 것이 전부이다. 
- Kernel_yield()를 호출한 태스크의 컨텍스트를 스택에 백업하고 스케줄러가 선정해 준 태스크의 스택 포인터를 복구한다. 그리고 스택 포인터로부터 컨텍스트를 복구한다. 

<br/>

# 10.4 커널 시작하기

- 커널을 시작할 때 현재 동작 중인 태스크가 없으므로 스케줄링을 하면 태스크가 동작하지 않는다. 스케줄링의 동작 순서는 현재 동작 중인 태스크가 있다는 것을 가정하고 있기 때문이다. 
- 해결 방법 : 최초로 스케줄링할 때는 컨텍스트 백업을 하지 않는다. 최초로 스케줄링할 때는 컨텍스트 복구만 하는 것이다. 최초 스케줄링이니까 스케줄러를 거치지 말고 그냥 0번 태스크 컨트롤 블록을 컨텍스트 복구 대상으로 삼는다. 

```bash
vi task.c
```

```c
static KernelTcb_t  sTask_list[MAX_TASK_NUM];
static KernelTcb_t* sCurrent_tcb;
static KernelTcb_t* sNext_tcb;
static uint32_t     sAllocated_tcb_index;
static uint32_t     sCurrent_tcb_index;

/… 중략 …/

void Kernel_task_init(void)
{
    sAllocated_tcb_index = 0;
    sCurrent_tcb_index = 0;

    for(uint32_t i = 0 ; i < MAX_TASK_NUM ; i++)
    {
        sTask_list[i].stack_base = (uint8_t*)(TASK_STACK_START + (i * USR_TASK_STACK_SIZE));
        sTask_list[i].sp = (uint32_t)sTask_list[i].stack_base + USR_TASK_STACK_SIZE - 4;

        sTask_list[i].sp -= sizeof(KernelTaskContext_t);
        KernelTaskContext_t* ctx = (KernelTaskContext_t*)sTask_list[i].sp;
        ctx->pc = 0;
        ctx->spsr = ARM_MODE_BIT_SYS;
    }
}

void Kernel_task_start(void)
{
    sNext_tcb = &sTask_list[sCurrent_tcb_index];
    Restore_context();
}

/… 후략 …/
```

- Kernel_task_init() 함수의 코드를 수정. 적당한 최초 스케줄링을 처리하는     Kernel_task_start() 함수 구현.

- sCurrent_tcb_index : 현재 실행 중인 태스크의 태스크 컨트롤 블록 인덱스를 저장하는 정적 전역 변수.

- Kernel_task_start() : 커널을 시작할 때 최초 한 번만 호출되는 함수. 
  - 스케줄러 호출은 생략되었으며, Kernel_task_start()는 커널이 시작할 때 한 번만 호출되는 함수이므로 해당 시점에 sCurrent_tcb_index 변수의 값은 반드시 초기값으로 설정한 0이어햐 한다. 
  - sNext_tcb에는 0번 태스크 컨트롤 블록의 포인터가 저장되며, 첫 번째로 생성된 태스크의 태스크 컨트롤 블록이 된다.
  - 이후 Restore_context() 함수를 호출한다. 컨텍스트 백업을 하지 않았으므로 지금까지의 컨텍스트는 모두 사라지고 태스크의 컨텍스트를 ARM 코어에 덮어 쓴다. 


```bash
vi Kernel.h
```

```c
#ifndef KERNEL_KERNEL_H_
#define KERNEL_KERNEL_H_

#include "task.h"

void Kernel_start(void);
void Kernel_yield(void);

#endif /* KERNEL_KERNEL_H_ */
```

- Kernel_start() 함수 정의. 
- Kernel_task_start() 함수를 커널 API인 Kernel_start() 함수에 연결.

>앞으로 추가할 커널 관련 초기화 함수를 Kernel_start() 함수에 모아서 한번에 실행한다. Kernel_start() 함수에서 커널 초기화를 담당하게 된다. 

```bash
vi Kernel.c
```

```c
#include "stdint.h"
#include "stdbool.h"

#include "Kernel.h"

void Kernel_start(void)
{
	Kernel_task_start();
}

void Kernel_yield(void)
{
	Kernel_task_scheduler();
}
```

- Kernel_start() 함수 구현.
- 그냥 Kernel_task_start() 함수를 직접 호출하는 것이 전부이다. 

```bash
vi Main.c
```

```c
#include "Kernel.h"

/… 중략 …/

static void Kernel_init(void)
{
    uint32_t taskId;

    Kernel_task_init();

    taskId = Kernel_task_create(User_task0);
    if (NOT_ENOUGH_TASK_NUM == taskId)
    {
        putstr("Task0 creation fail\n");
    }

    taskId = Kernel_task_create(User_task1);
    if (NOT_ENOUGH_TASK_NUM == taskId)
    {
        putstr("Task1 creation fail\n");
    }

    taskId = Kernel_task_create(User_task2);
    if (NOT_ENOUGH_TASK_NUM == taskId)
    {
        putstr("Task2 creation fail\n");
    }

    Kernel_start();
}
```

- Kernel_start() 함수를 호출하는 코드를 main() 함수에 추가하고 QEMU를 실행해서 스케줄링 동작을 확인한다.
- Kernel_start() 함수를 호출하게 되면 0번 태스크에 등록된 함수가 실행된다. 현재     User_task0() 함수를 0번 태스크로 등록했으므로 해당 태스크가 실행 될 것이다.

```c
// Main.c
void User_task0(void);
void User_task1(void);
void User_task2(void);

/… 중략 …/
       
void User_task0(void)
{
    uint32_t local = 0;

    while(true)
    {
	debug_printf("User Task #0 SP=0x%x\n", &local);
	Kernel_yield();
    }
}

void User_task1(void)
{
    uint32_t local = 0;

    while(true)
    {
	debug_printf("User Task #1 SP=0x%x\n", &local);
	Kernel_yield();
    }
}

void User_task2(void)
{
    uint32_t local = 0;

    while(true)
    {
	debug_printf("User Task #2 SP=0x%x\n", &local);
	Kernel_yield();
    }
}
```

- 사용자 태스크가 제대로 스택을 할당받았는지 확인해 보는 코드를 추가해서 우리가 의도한 모든 것이 잘되었는지를 확인해 본다.
- 각 태스크에 등록된 함수 내에서 선언된 로컬 변수의 주소를 읽어서 각 태스크의 스택 주소를 확인한다.
- 아무 로컬 변수를 하나 선언한 뒤 이전에 만들었는 훌륭한 디버그 도구인 debug_printf() 함수를 이용하여 그 변수의 주소 값을 출력한다.
- while 무한 루프문을 이용하여 Kernel_yield() 함수를 계속해서 호출한다. 커널이 의도한 대로 동작한다면 새 태스크의 로컬 변수 주소(스택 주소)가 반복적으로 출력되며 멈추지 않고 동작할 것이다. 

```bash
make run
```

- 종료되지 않고 끝없이 반복하며 로컬 변수의 주소가 출력된다면 Kernel_start()와 Kernel_yield() 함수가 잘 동작하는 것을 확인할 수 있다.

- 각 태스크의 스택이 제대로 할당되었는지 값을 확인해본다. 
  - TASK_STACK_START의 값이 0x80000000이다. 이 값은 Task#0의 스택 베이스 주소이다.
  - 스택 포인터에는 스택 공간의 최댓값을 할당한다. 
  - 태스크 스택 간에 4바이트 간격을 패딩으로 설계하였다.
  - 여기에 컴파일러가 사용하는 스택이 몇 개 되고 그 다음에 로컬 변수가 스택에 잡히므로 스택 메모리 주소가 0x8FFFF0으로 출력된 것이다.
  - 마찬가지로 Task#1과 Task#2는 각각 1MB씩의 간격을 차이로 출력된다.
