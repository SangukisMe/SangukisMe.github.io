---
title: "임베디드 OS 개발 프로젝트 13장 :: SYNCHRONIZATION(2)"
excerpt: "뮤텍스"

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

# 13.2 뮤텍스

- 세마포어 : 잠금에 대한 소유 개념이 없으므로 누가 잠근 세마포어이든 간에 누구나 잠금을 풀 수 있다.
- 뮤텍스 : 바이너리 세마포어에 소유의 개념을 더한 동기화 알고리즘. 뮤텍스를 잠근 태스크만이 뮤텍스의 잠금을 풀 수 있다.

## STEP1. 뮤텍스 구현

```bash
vi synch.h
```

```c
#ifndef KERNEL_SYNCH_H_
#define KERNEL_SYNCH_H_

typedef struct KernelMutext_t
{
    uint32_t owner;
    bool     lock;
} KernelMutext_t;

void Kernel_sem_init(int32_t max);
bool Kernel_sem_test(void);
void Kernel_sem_release(void);

void Kernel_mutex_init(void);
bool Kernel_mutex_lock(uint32_t owner);
bool Kernel_mutex_unlock(uint32_t owner);

#endif /* KERNEL_SYNCH_H_ */
```

- 뮤텍스 관련 자료 구조와 함수 프로토타입을 선언.
- KernelMutext_t : 뮤텍스의 소유자와 잠김을 표시하는 변수를 추상화한 구조체.
- 세마포어와 다르게 뮤텍스 함수에서 파라미터로 소유자(owner)를 받는다.

```bash
vi synch.c
```

```c
KernelMutext_t sMutex;

/… 중략 …/

void Kernel_mutex_init(void)
{
    sMutex.owner = 0;
    sMutex.lock = false;
}

bool Kernel_mutex_lock(uint32_t owner)
{
    if (sMutex.lock)
    {
        return false;
    }

    sMutex.owner = owner;
    sMutex.lock = true;
    return true;
}

bool Kernel_mutex_unlock(uint32_t owner)
{
    if (owner == sMutex.owner)
    {
        sMutex.lock = false;
        return true;
    }
    return false;
}
```

- sMutex : 뮤텍스 자료 구조를 전역 변수로 구현. 이 전역 변수로 커널 뮤텍스를 제어한다. 필요에 따라 배열로 만들어 뮤텍스를 여러 개 사용할 수도 있다.
- Kernel_mutex_init() : 전역 변수 sMutex를 0으로 초기화 하는 함수. 뮤텍스는 바이너리 세마포어의 일종이므로 잠금 개수는 한 개 이다. 따라서 별도로 최댓값 파라미터를 받지 않는다.
- Kernel_mutex_lock() : 뮤텍스를 잠그는 함수. 세마포어와 비슷하게 뮤텍스가 잠겨있으면 false를 리턴한다. 잠겨있지 않다면 소유자를 등록하고 뮤텍스를 잠근다.
- Kernel_mutex_unlock() : 뮤텍스의 잠금을 푸는 함수. 세마포어와의 차이점이 확실히 나타나는 함수이다. 소유자를 확인하고 뮤텍스를 잠갔던 소유자일 때만 뮤텍스의 잠금 해제를 허용한다. 소유자가 아닌 태스크에서 뮤텍스의 잠금 해제를 요청하면 무시하고 false를 리턴한다.

<br/>

## STEP2. 뮤텍스 커널 API 구현

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
void 			  Kernel_lock_mutex(void);
void 		 	  Kernel_unlock_mutex(void);

#endif /* KERNEL_KERNEL_H_ */
```

- 뮤텍스의 커널 API 선언.

```bash
vi Kernel.c
```

```c
void Kernel_lock_mutex(void)
{
	while(true)
	{
		uint32_t current_task_id = Kernel_task_get_current_task_id();
		if (false == Kernel_mutex_lock(current_task_id))
		{
			Kernel_yield();
		}
		else
		{
			break;
		}
	}
}

void Kernel_unlock_mutex(void)
{
	uint32_t current_task_id = Kernel_task_get_current_task_id();
	if (false == Kernel_mutex_unlock(current_task_id))
	{
		Kernel_yield();
	}
}
```

- Kernel_lock_mutex() : 뮤텍스를 잠그는(lock) 함수. 현재 CPU를 점유중인 태스크를 뮤텍스의 소유주로 등록하고 뮤텍스를 잠근다. 세마포어와 비슷하게 뮤텍스가 잠겨있을 경우 스케줄링을 해준다. 
- Kernel_unlock_mutex() : 뮤텍스를 푸는(unlock) 함수. 현재 CPU를 점유중인 태스크와 뮤텍스의 소유주와 비교해서 소유주가 아닐 경우 뮤텍스 해제에 실패하게 되고 스케줄링을 해서 소유주가 잠금을 풀 수 있도록 한다.

>```bash
>vi task.c
>```
>
>```c
>uint32_t Kernel_task_get_current_task_id(void)
>{
>	return sCurrent_tcb_index;
>}
>```
>
>- Kernel_task_get_current_task_id() : 현재 동작 중인 태스크의 태스크 ID를 리턴하는 함수.
>
>태스크에 관련한 함수이므로 task.c 파일에 추가한다. task.h 파일에 프로토타입을 선언해준다.

<br/>

## STEP3. 세마포어와 뮤텍스의 차이를 확인하기 위한 코드 구현

```bash
vi event.h
```

```c
typedef enum KernelEventFlag_t
{
    KernelEventFlag_UartIn      = 0x00000001,
    KernelEventFlag_CmdIn       = 0x00000002,
    KernelEventFlag_CmdOut      = 0x00000004,
    KernelEventFlag_Unlock        = 0x00000008,
    KernelEventFlag_Reserved04  = 0x00000010,
    KernelEventFlag_Reserved05  = 0x00000020,
    KernelEventFlag_Reserved06  = 0x00000040,
    KernelEventFlag_Reserved07  = 0x00000080,
    KernelEventFlag_Reserved08  = 0x00000100,
    KernelEventFlag_Reserved09  = 0x00000200,

/… 중략 …/
```

- 새로운 이벤트를 Unlock이라는 이름으로 추가.

```bash
vi Uart.c
```

```c
static void interrupt_handler(void)
{
           uint8_t ch = Hal_uart_get_char();
   
	if (ch == 'U')
	{
		Kernel_send_events(KernelEventFlag_Unlock);
		return;
	}

	if (ch == 'X')
	{
    	        Kernel_send_events(KernelEventFlag_CmdOut);
	            return;
	}

	Hal_uart_put_char(ch);
           Kernel_send_msg(KernelMsgQ_Task0, &ch, 1);
           Kernel_send_events(KernelEventFlag_UartIn);
}
```

- 뮤텍스의 기능을 확인하기 위해 UART 인터럽트 핸들러 코드를 수정한다.
- 나머지 코드는 기존과 동일하다. 키보드 대문자 U키를 입력했을 때 KernelEventFlag_Unlock 이벤트가 발생한다. Unlock 이벤트는 Task1에서 받아서 처리하도록 한다.

```bash
vi Main.c
```

```c
void User_task1(void)
{
    uint32_t local = 0;

    debug_printf("User Task #1 SP=0x%x\n", &local);

    uint8_t cmdlen = 0;
    uint8_t cmd[16] = {0};

    while(true)
    {
        KernelEventFlag_t handle_event = Kernel_wait_events(KernelEventFlag_CmdIn|KernelEventFlag_Unlock);
        switch(handle_event)
        {
        case KernelEventFlag_CmdIn:
            memclr(cmd, 16);
            Kernel_recv_msg(KernelMsgQ_Task1, &cmdlen, 1);
            Kernel_recv_msg(KernelMsgQ_Task1, cmd, cmdlen);
            debug_printf("\nRecv Cmd: %s\n", cmd);
            break;
	case KernelEventFlag_Unlock:
		Kernel_unlock_sem();
		break;
        }
        Kernel_yield();
    }
}
```

- Task1에서 Unlock 이벤트도 기다릴 수 있도록 Kernel_wait_events() 커널 API에 보내는 파라미터를 수정한다.
- Unlock 이벤트 핸들러에 세마포어를 해제하는 Kernel_unlock_sem() 커널 API만 호출한다. 즉, 키보드 대문자 U를 입력하면 Task1은 그냥 다짜고짜 세마포어를 해제하는 것이다.

```bash
// Main.c

static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    Kernel_lock_sem();

    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    Kernel_yield();
    delay(1000);
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);

   // Kernel_unlock_sem();
}
```

- 크리티컬 섹션 함수에서 세마포어를 해제하는 함수를 제거하였다. 크리티컬 섹션에 진입하면서 세마포어를 잠그기만하고 해제는 하지 않는 것이다.

```bash
make run
```

- 동작 확인
  - User Task #2 Send=3
  - User Task #2 Shared Value=3
  - 여기서 멈춘다. U 키를 누르면 다시 위 두 문자를 출력하고 다시 멈춘다.

- 크리티컬 섹션 함수를 호출한 태스크는 자신이 잠갔던 세마포어에 걸려서 크리티컬 섹션에 진입하지 못하게 된다. 이 상태에서 U 키를 입력하면 Task1이 세마포어를 풀어서 Task2가 크리티컬 섹션을 한 번 실행하고 다시 멈추게 되는 것이다.

- 세마포어를 잠그는 주체와 푸는 주체가 달라도 상관이 없는 것이다. 잠그고 푸는 횟수와 순서만 맞으면 된다.

```bash
vi Main.c
```

```c
static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    Kernel_lock_mutex();

    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    Kernel_yield();
    delay(1000);
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);
}

static void Kernel_init(void)
{
    uint32_t taskId;

    Kernel_task_init();
    Kernel_event_flag_init();
    Kernel_msgQ_init();
    Kernel_sem_init(1);
   Kernel_mutex_init();

    /…중략 …/

}

void User_task1(void)
{
    /… 중략 …/

	case KernelEventFlag_Unlock:
		Kernel_unlock_mutex();
		break;
        }
        Kernel_yield();
    }
}
```

- Kernel_lock_mutex()와 Kernel_unlock_mutex() 커널 API를 호출하는 코드로 수정.

```bash
make run
```

- 동작 확인
  - User Task #2 Send=3
  - User Task #2 Shared Value=3
  - 여기서 멈춘다. 아무리 U키를 눌러도 아무런 출력이 나오지 않는다.

- 뮤텍스를 잠근 태스크는 Task2이므로 Task1에서 아무리 Kernel_unlock_mutex() 커널 API를 호출해 봤자 뮤텍스는 풀리지 않는다.

## STEP4. 뮤텍스 적용 및 동작 확인

```bash
vi Main.c
```

```c
static uint32_t shared_value;
static void Test_critical_section(uint32_t p, uint32_t taskId)
{
    Kernel_lock_mutex();

    debug_printf("User Task #%u Send=%u\n", taskId, p);
    shared_value = p;
    Kernel_yield();
    delay(1000);
    debug_printf("User Task #%u Shared Value=%u\n", taskId, shared_value);

  Kernel_unlock_mutex();
}
```

- 뮤텍스의 잠금과 해제를 다시 원래 목적에 맞게 크리티컬 섹션의 들어오는 지점과 나가는 지점에 호출한다.

```bash
make run
```

- 대문자 X를 입력하여 Task0이 크리티컬 섹션에 끼어들게 만들어도 뮤텍스로 크리티컬 섹션을 보호하고 있으므로 Task2가 잠근 뮤텍스는 Task2가 풀고 나오고, Task0이 잠근 뮤텍스는 Task0이 풀고 나온다.

