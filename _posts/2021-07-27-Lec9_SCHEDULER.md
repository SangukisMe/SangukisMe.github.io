---
title: "임베디드 OS 개발 프로젝트 9장 :: SCHEDULER"
excerpt: "간단한 스케줄러 설계"

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

# 9장 tree

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

# 9.1 간단한 스케줄러

- 스케줄러란 지금 실행 중인 태스크 다음에 실행할 태스크가 무엇인지 골라주는 녀석이다. 스케줄러를 얼마나 효율적으로 만드느냐에 따라서 RTOS의 성능이 좌우되기도 할 정도로 중요하다.

```bash
vi task.c
```

```c
static uint32_t     sCurrent_tcb_index;
static KernelTcb_t* Scheduler_round_robin_algorithm(void);

/… 중략 …/

static KernelTcb_t* Scheduler_round_robin_algorithm(void)
{
    sCurrent_tcb_index++;
    sCurrent_tcb_index %= sAllocated_tcb_index;

    return &sTask_list[sCurrent_tcb_index];
}
```

- 인덱스를 계속 증가시키면서 대상을 선택하는 라운드 로빈(round robin) 알고리즘을 적용한 스케줄러 함수 구현.
- sCurrent_tcb_index : 현재 실행 중인 태스크의 태스크 컨트롤 블록 인덱스를 저장. 스케줄러 함수가 호출되면 인덱스를 증가시켜 다음에 동작할 태스크 컨트롤 블록 인덱스로 만든다. 현재 생성된 태스크 컨트롤 블록 인덱스의 최대값을 넘게되면 값이 0이 된다.
- sTask_list 배열을 읽어 다음에 동작할 태스크 컨트롤 블록을 리턴. 

