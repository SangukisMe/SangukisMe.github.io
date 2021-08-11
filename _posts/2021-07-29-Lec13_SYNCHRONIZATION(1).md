---
title: "임베디드 OS 개발 프로젝트 13장 :: SYNCHRONIZATION(1)"
excerpt: "세마포어"

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

# 13장 tree

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
│   ├── Kernel.c  
│   ├── Kernel.h  
│   ├── event.c  
│   ├── event.h  
│   ├── msg.c  
│   ├── msg.h  
│   ├── **synch.c**  
│   ├── **synch.h**  
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

# 13.1 세마포어

- 아토믹 오퍼레이션 : 해당 작업이 끝날 때까지 컨텍스트 스위칭이 발생하지 않는 상태.
- 크리티컬 섹션 : 어떤 작업이 아토믹하게 구현되어야만 하는 작업.
- 동기화 : 어떤 작업이 크리티컬 섹션 이라고 판단되었을 경우, 해당 크리티컬 섹션을 아토믹 오퍼레이션으로 만들어 주는 것.

## STEP1. 세마포어 구현

```bash
vi synch.h
```

```c
#ifndef KERNEL_SYNCH_H_
#define KERNEL_SYNCH_H_

void Kernel_sem_init(int32_t max);
bool Kernel_sem_test(void);
void Kernel_sem_release(void);

#endif /* KERNEL_SYNCH_H_ */
```

- 세마포어를 구현하기 위한 함수 선언

```bash
vi synch.c
```

```c
#include "stdint.h"
#include "stdbool.h"

#include "synch.h"

#define DEF_SEM_MAX 8

static int32_t sSemMax;
static int32_t sSem;

KernelMutext_t sMutex;

void Kernel_sem_init(int32_t max)
{
    sSemMax = (max <= 0) ? DEF_SEM_MAX : max;
    sSemMax = (max >= DEF_SEM_MAX) ? DEF_SEM_MAX : max;

    sSem = sSemMax;
}

bool Kernel_sem_test(void)
{
    if (sSem <= 0)
    {
        return false;
    }

    sSem--;

    return true;
}

void Kernel_sem_release(void)
{
    sSem++;

    if (sSem >= sSemMax)
    {
        sSem = sSemMax;
    }
}
```

- Kernel_sem_init() : 세마포어 초기화 함수. Max 파라미터로 세마포어의 최댓값을 받는다. 세마포어의 개수를 1부터 8까지 지정할 수 있도록 한다.
- Kernel_sem_test() : 크리티컬 섹션에 진입 가능한지를 확인해 보는 함수. 세마포어를 잠글(lock) 수 있는지 확인한다. 세마포어를 잠글 수 없을 때 대기하는 대신 false를 리턴.
- Kernel_sem_release() : 크리티컬 섹션을 나갈 때 호출하는 함수. 세마포어의 잠금을 푸는(unlock) 역할을 한다. 정해놓은 세마포어의 최댓값을 넘지 않도록 조정한다.

>바이너리 세마포어 : 세마포어의 개수가 1개인 것을 바이너리 세마포어라고 한다. 
>
>세마포어를 잠글 수 없다면 false 리턴 : 싱글 코어 환경에서 세마포어가 잠겨있을경우 다른 태스크에서 해당 세마포어를 풀어주지 않으면 영원히 대기하게 된다. 따라서 다른 태스크로 스위칭하여 세마포어를 풀어주고 다시 세마포어를 획득하도록 한다. 해당 코드는 커널 API 함수에서 구현하도록 한다. 

<br/>

## STEP2. 세마포어 커널 API 구현

```bash
vi Kernel.h
```

```c
#ifndef KERNEL_KERNEL_H_
#define KERNEL_KERNEL_H_

#include "task.h"
#include "event.h"
#include "msg.h"
#include "synch.h"

void              Kernel_start(void);
void              Kernel_yield(void);
void              Kernel_send_events(uint32_t event_list);
KernelEventFlag_t Kernel_wait_events(uint32_t waiting_list);
bool              Kernel_send_msg(KernelMsgQ_t Qname, void* data, uint32_t count);
uint32_t          Kernel_recv_msg(KernelMsgQ_t Qname, void* out_data, uint32_t count);
void 			  Kernel_lock_sem(void);
void			  Kernel_unlock_sem(void);

#endif /* KERNEL_KERNEL_H_ */
```

- 세마포어의 커널 API 선언.

```bash
vi Kernel.c
```

```c
#include "stdint.h"
#include "stdbool.h"

#include "memio.h"
#include "Kernel.h"

/… 중략 …/

void Kernel_lock_sem(void)
{
	while(false == Kernel_sem_test())
	{
		Kernel_yield();
	}
}

void Kernel_unlock_sem(void)
{
	Kernel_sem_release();
}
```

- Kernel_lock_sem() : 세마포어를 잠그는(lock) 함수. 세마포어가 이미 잠겨있다면 해당 크리티컬 섹션의 잠금을 소유하고 있는 다른 태스크로 컨텍스트가 넘어가서 세마포어의 잠금을 풀어줄 수 있도록 스케줄링(Kernel_yield()) 함수를 호출한다. 즉, 세마포어를 획득하기 전까지 해당 태스크는 Kernel_lock_sem() 함수를 빠져나오지 않는다. 해당 코드는 멀티태스킹에서 대기(waiting)를 어떻게 구현하는지 보여준다.
- Kernel_unlock_sem() : 세마포어를 푸는(unlock) 함수. 그저 Kernel_sem_release() 함수를 호출한다.

<br/>

## STEP3. 동기화 문제를 발생시키기 위한 코드 구현

```bash
vi Uart.c
```

```c
static void interrupt_handler(void)
{
           uint8_t ch = Hal_uart_get_char();
   
	if (ch != 'X')
	{
		Hal_uart_put_char(ch);
    	       Kernel_send_msg(KernelMsgQ_Task0, &ch, 1);
    	       Kernel_send_events(KernelEventFlag_UartIn);
	}
	else
	{
    	       Kernel_send_events(KernelEventFlag_CmdOut);
	}
}
```

- 세마포어의 기능을 확인하기 위해 UART 인터럽트 핸들러 코드를 수정한다.
- 기존 코드는 살려두고 조건을 추가하여 키보드에서 대문자 X를 입력했을 때 다른 동작을 하도록 코드 수정. 대문자 X를 입력하면 KernelEventFlag_CmdOut 이벤트가 발생한다.

>나빌로스는 비선점형 스케줄러인데다가 커널이 강제로 스케줄링을 하는 것이 아니라 태스크가 Kernel_yield() 함수를 호출해야만 스케줄링이 동작하므로 동기화 문제가 발생하는 코드를 만드는 것이 더 어렵다.

```bash
vi Main.c
```

```c
static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    Kernel_yield();
    delay(1000);
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);
}

/… 중략 …/

void User_task0(void)
{
    uint32_t local = 0;
    debug_printf("User Task #0 SP=0x%x\n", &local);

    uint8_t  cmdBuf[16];
    uint32_t cmdBufIdx = 0;
    uint8_t  uartch = 0;

    while(true)
    {
        KernelEventFlag_t handle_event = Kernel_wait_events(KernelEventFlag_UartIn|KernelEventFlag_CmdOut);
        switch(handle_event)
        {
        case KernelEventFlag_UartIn:
            Kernel_recv_msg(KernelMsgQ_Task0, &uartch, 1);
            if (uartch == '\r')
            {
                cmdBuf[cmdBufIdx] = '\0';

                Kernel_send_msg(KernelMsgQ_Task1, &cmdBufIdx, 1);
                Kernel_send_msg(KernelMsgQ_Task1, cmdBuf, cmdBufIdx);
                Kernel_send_events(KernelEventFlag_CmdIn);

                cmdBufIdx = 0;
            }
            else
            {
                cmdBuf[cmdBufIdx] = uartch;
                cmdBufIdx++;
                cmdBufIdx %= 16;
            }
            break;
        case KernelEventFlag_CmdOut:
            Test_critical_section(5, 0);
            break;
        }
        Kernel_yield();
    }
}
```

- Test_critical_section() : 크리티컬 섹션 함수. 크리티컬 섹션을 만들기 위해 여러 태스크 혹은 여러 코어가 공유하는 공유 자원 변수의 값을 함수 내에서 사용한다. 중간에 Kernel_yield() 함수를 호출해서 억지로 스케줄링을 한다.

  - uint32_t p : 공유 자원(shared_value)의 값을 바꿀 입력 값.
  - uint32_t taskId : 함수를 호출한 태스크의 번호. 

>해당 코드는 공유 자원 문제를 만들기 위해 만든 억지스러운 코드이다. Test_critical_section이라는 함수를 이용해 크리티컬 섹션 상태에서의 공유 자원의 값이 원치않는 값으로 바뀌는 것을 확인해 본다. 

```bash
make run
```

- X 키를 누를 때마다 Task0이 숫자 5를 shared_value 변수에 전달하는 것을 디버그 메시지로 출력한다.
  - User Task #0 Send=5 : Task0이 숫자 5를 전달.
  - User Task #0 Shared Value=5 : Task0에서 출력한 공유 변수의 값이 5다.

- Test_critical_section() 함수는 반드시 입력으로 전달한 값과 공유 변수의 값이 같아야만 제 역할을 했다고 볼 수 있다.

```c
// Main.c

void User_task2(void)
{
    uint32_t local = 0;

    debug_printf("User Task #2 SP=0x%x\n", &local);

    while(true)
    {
        Test_critical_section(3, 2);
        Kernel_yield();
    }
}
```

- Test_critical_section() 함수를 또 다른 태스크인 Task2에서 동시에 호출하도록 구현.
- Task2에서는 공유 자원에 입력값으로 3을 전달한다. 

```bash
make run
```

- 정상적으로 실행 될 경우 Task2의 출력이 계속 나온다.

- X키를 누르게 되면 Task2가 Task0이 shared_value에 5를 넣는 동작을 방해하는 형태가 된다.

  - User Task #0 Send=5 : Task0이 숫자 5를 전달.
  - User Task #2 Send=3 : Task0이 숫자 5를 전달.
  - User Task #0 Shared Value=3 : Task0에서 출력한 공유 변수의 값이 3이다.
  - Task0은 5를 보냈으므로 위와 같은 출력이 나오는 것은 분명히 잘못된 결과이다. 이런 결과가 생긴 이유는 Test_critical_section() 함수 중간에 억지로 호출한 Kerenl_yield() 함수 때문이다. Task0이 바꾼 shared_value의 값을 출력하기 전에 Task1에서 shared_value의 값을 바꿔버리는 것이다.

>멀티코어 환경에서 크리티컬 섹션에 이와 비슷한 성격의 공유 자원 문제는 매우 자주 발생한다. 여러 코어가 공유하는 자원에 대한 값을 바꾸고 사용하는 코드라면 개발자가 판단해서 이것을 크리티컬 섹션으로 식별하고 반드시 동기화 처리를 해야만 한다.

<br/>

## STEP4. 세마포어 적용 및 동작 확인

```bash
vi Main.c
```

```c
static void Kernel_init(void)
{
    uint32_t taskId;

    Kernel_task_init();
    Kernel_event_flag_init();
    Kernel_msgQ_init();
    Kernel_sem_init(1);

   /… 중략 …/

}

static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    Kernel_lock_sem();

    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    Kernel_yield();
    delay(1000);
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);

    Kernel_unlock_sem();
}
```

- 바이너리 세마포어를 만들어서 크리티컬 섹션에 동기화 처리를 한다.
- Kernel_sem_init(1) : 세마포어의 잠금 개수를 1로 설정. 바이너리 세마포어를 만든다.
- Kernel_lock_sem() : 크리티컬 섹션에 집입할 때 호출.
- Kernel_unlock_sem() : 크리티컬 섹션이 끝나면 호출.
- 두 개의 커널 API를 크리티컬 섹션의 처음과 끝에 호출함으로써 Test_critical_section 함수의 동작을 아토믹 오퍼레이션으로 만든다.

```bash
make run
```

- 정상적으로 동기화가 되었다면 Task0과 Task2에서 입력한 공유 자원의 값이 다른 태스크에 의해 방해받지 않는 것을 확인할 수 있다.
