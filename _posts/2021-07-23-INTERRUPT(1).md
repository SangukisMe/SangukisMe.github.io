---
title: "임베디드 OS 개발 프로젝트 6장 :: INTERRUPT(1)"
excerpt: "인터럽트 컨트롤러"

toc: true # Table of Contents
toc_sticky: true # TOC를 고정해주는 역할 
# toc_label: # TOC의 제목 설정

categories:
  - Embedded RTOS
tags:
  - Embedded
  - RTOS
last_modified_at: 2021-07-23
---

<br/>

# 6장 tree

.  
├── Makefile  
├── boot  
│   ├── Entry.S  
│   ├── **Handler.c**  
│   └── Main.c  
├── hal  
│   ├── **HalInterrupt.h**  
│   ├── HalUart.h  
│   └── rvpb  
│       ├── **Interrupt.c**  
│       ├── **Interrupt.h**  
│       ├── Regs.c  
│       ├── Uart.c  
│       └── Uart.h  
├── include  
│   ├── ARMv7AR.h  
│   ├── MemoryMap.h  
│   ├── **memio.h**  
│   ├── stdarg.h  
│   └── stdint.h  
├── lib  
│   ├── **armcpu.c**  
│   ├── **armcpu.h**  
│   ├── stdio.c  
│   └── stdio.h  
└── navilos.ld

<br/>

# 6.1 인터럽트 컨트롤러

```bash
vi Interrupt.h
```

```c
#ifndef HAL_RVPB_INTERRUPT_H_
#define HAL_RVPB_INTERRUPT_H_

typedef union CpuControl_t
{
    uint32_t all;
    struct {
        uint32_t Enable:1;          // 0
        uint32_t reserved:31;
    } bits;
} CpuControl_t;

…  중략 ...

typedef struct GicCput_t
{
    CpuControl_t       cpucontrol;        //0x000
    PriorityMask_t     prioritymask;      //0x004
    BinaryPoint_t      binarypoint;       //0x008
    InterruptAck_t     interruptack;      //0x00C
    EndOfInterrupt_t   endofinterrupt;    //0x010
    RunningInterrupt_t runninginterrupt;  //0x014
    HighestPendInter_t highestpendinter;  //0x018
} GicCput_t;

typedef struct GicDist_t
{
    DistributorCtrl_t   distributorctrl;    //0x000
    ControllerType_t    controllertype;     //0x004
    uint32_t            reserved0[62];      //0x008-0x0FC
    uint32_t            reserved1;          //0x100
    uint32_t            setenable1;         //0x104
    uint32_t            setenable2;         //0x108
    uint32_t            reserved2[29];      //0x10C-0x17C
    uint32_t            reserved3;          //0x180
    uint32_t            clearenable1;       //0x184
    uint32_t            clearenable2;       //0x188
} GicDist_t;

#define GIC_CPU_BASE  0x1E000000  //CPU interface
#define GIC_DIST_BASE 0x1E001000  //distributor

#define GIC_PRIORITY_MASK_NONE  0xF

#define GIC_IRQ_START           32
#define GIC_IRQ_END             95

#endif /* HAL_RVPB_INTERRUPT_H_ */
```

- GIC(Generic Interrupt Controller) 레지스터 구조체 
- GIC 레지스터를 'CPU Interface registers'와 'Distributor registers' 두 개의 레지스터로 구분

```bash
vi Regs.c
```

```c
#include "stdint.h"
#include "Uart.h"
#include "Interrupt.h"

volatile PL011_t* Uart = (PL011_t*)UART_BASE_ADDRESS0;D
volatile GicCput_t* GicCpu = (GicCput_t*)GIC_CPU_BASE;
volatile GicDist_t* GicDist = (GicDist_t*)GIC_DIST_BASE;
```

- GIC 레지스터 구조체의 제어 인스턴스 선언
- GIC 구조체 포인터 변수를 선언하고 베이스 주소 할당.

```bash
vi HalInterrupt.h
```

```c
#ifndef HAL_HALINTERRUPT_H_
#define HAL_HALINTERRUPT_H_

#define INTERRUPT_HANDLER_NUM   255

typedef void (*InterHdlr_fptr)(void);

void Hal_interrupt_init(void);
void Hal_interrupt_enable(uint32_t interrupt_num);
void Hal_interrupt_disable(uint32_t interrupt_num);
void Hal_interrupt_register_handler(InterHdlr_fptr handler, uint32_t interrupt_num);
void Hal_interrupt_run_handler(void);

#endif /* HAL_HALINTERRUPT_H_ */
```

- GIC에 대한 공용 API 함수 선언.
- Hal_interrupt_init : 인터럽트 초기화 함수
- Hal_interrupt_enable : 인터럽트 활성화 함수 
- Hal_interrupt_disable : 인터럽트 비활성화 함수
- Hal_interrupt_register_handler : 인터럽트 핸들러 등록 함수
- Hal_interrupt_run_handler : 인터럽트 핸들러 호출 함수

>ARM은 모든 인터럽트를 IRQ 혹은 FIQ 핸들러로 처리. IRQ나 FIQ 핸들러 내에서 개별 인터럽트의 핸들러를 구분해야 한다. 이러한 개별 인터럽트를 등록하고 실행하는 역할을 하는 함수이다.

```bash
vi Interrupt.c
```

```c
#include "stdint.h"
#include "memio.h"
#include "Interrupt.h"
#include "HalInterrupt.h"
#include "armcpu.h"

extern volatile GicCput_t* GicCpu;
extern volatile GicDist_t* GicDist;

static InterHdlr_fptr sHandlers[INTERRUPT_HANDLER_NUM];

void Hal_interrupt_init(void)
{
    GicCpu->cpucontrol.bits.Enable = 1;
    GicCpu->prioritymask.bits.Prioritymask = GIC_PRIORITY_MASK_NONE;
    GicDist->distributorctrl.bits.Enable = 1;

    for (uint32_t i = 0 ; i < INTERRUPT_HANDLER_NUM ; i++)
    {
        sHandlers[i] = NULL;
    }

    enable_irq();
}
```

- Hal_interrupt_init 함수 구현
- CPU interface와 Distributor 레지스터에 접근해 인터럽트 컨트롤러를 On 한다.
- CPU interface의 Priority mask 레지스터에 접근해 모든 인터럽트를 허용한다. 
- enable_irq() : ARM의 cspr(상태 레지스터)을 제어해서 코어 수준의 IRQ를 On 하는 함수(armcpu.h / armcpu.c 파일에 정의 및 구현되있다.) 

>sHandlers : 인터럽트 핸들러를 저장해 놓을 변수. INTERRUPT_HANDLER_NUM을 255로 선언했으므로 sHandlers는 함수 포인터 255개를 저장할 수 있는 배열이다. 가능하면 프로젝트에 필요한 인터럽트 핸들러 개수에 맞추어 설정하는 것이 좋다.

```c
// Interrupt.c 

void Hal_interrupt_enable(uint32_t interrupt_num)
{
    if ((interrupt_num < GIC_IRQ_START) || (GIC_IRQ_END < interrupt_num))
    {
        return;
    }

    uint32_t bit_num = interrupt_num - GIC_IRQ_START;

    if (bit_num < GIC_IRQ_START)
    {
        SET_BIT(GicDist->setenable1, bit_num);
    }
    else
    {
        bit_num -= GIC_IRQ_START;
        SET_BIT(GicDist->setenable2, bit_num);
    }
}

void Hal_interrupt_disable(uint32_t interrupt_num)
{
    if ((interrupt_num < GIC_IRQ_START) || (GIC_IRQ_END < interrupt_num))
    {
        return;
    }

    uint32_t bit_num = interrupt_num - GIC_IRQ_START;

    if (bit_num < GIC_IRQ_START)
    {
        CLR_BIT(GicDist->setenable1, bit_num);
    }
    else
    {
        bit_num -= GIC_IRQ_START;
        CLR_BIT(GicDist->setenable2, bit_num);
    }
}
```

- Hal_interrupt_enable, Hal_interrupt_disable 함수 구현
- GIC는 64개의 인터럽트를 관리할 수 있다. Set Enable1, Set Enable2 두 개의 레지스터에 각각 32개씩 할당해 놓는다.
- IRQ ID 32 ~ 63 : Set Enable1 레지스터의 (ID - 32)번 bit에 1(On) / 0(Off) 설정.
- IRQ ID 64 ~ 95 : Set Enable2 레지스터의 (ID - 32 - 32)번 bit에 1(On) / 0(Off) 설정.
- SET_BIT와 CLR_BIT라는 매크로를 이용하여 설정(memio.h 파일에 구현)

```c
// Interrupt.c 
 
void Hal_interrupt_register_handler(InterHdlr_fptr handler, uint32_t interrupt_num)
{
    sHandlers[interrupt_num] = handler;
}

void Hal_interrupt_run_handler(void)
{
    uint32_t interrupt_num = GicCpu->interruptack.bits.InterruptID;

    if (sHandlers[interrupt_num] != NULL)
    {
        sHandlers[interrupt_num]();
    }

    GicCpu->endofinterrupt.bits.InterruptID = interrupt_num;
}
```

- Hal_interrupt_register_handler, Hal_interrupt_run_handler 함수 구현

- Hal_interrupt_register_handler() : IRQ ID 번호를 기준으로 개별 인터럽트 핸들러를 sHandlers에 등록

- Hal_interrupt_run_handler()
  - Interrupt acknowledge 레지스터에서 현재 하드웨어에서 대기 중인 인터럽트 IRQ ID 번호를 읽어온다.
  - 읽어온 IRQ ID 번호를 sHandlers의 인덱스로 사용하여 sHandlers에 저장된 인터럽트 핸들러 함수 포인터를 실행한다.
  - 인터럽트 처리가 끝나면 End or interrupt 레지스터에 IRQ ID를 써넣어 인터럽트 컨트롤러에 해당 인터럽트에 대한 처리가 끝났다는 것을 알려준다.


```bash
vi memio.h
```

```c
#ifndef INCLUDE_MEMIO_H_
#define INCLUDE_MEMIO_H_

#define SET_BIT(p,n) ((p) |=  (1 << (n)))
#define CLR_BIT(p,n) ((p) &= ~(1 << (n)))

#endif /* INCLUDE_MEMIO_H_ */
```

```bash
vi armcpu.h
```

```c
#ifndef LIB_ARMCPU_H_
#define LIB_ARMCPU_H_

void enable_irq(void);
void enable_fiq(void);
void disable_irq(void);
void disable_fiq(void);

#endif /* LIB_ARMCPU_H_ */
```

- ARM의 cspr의 IRQ/FIQ 마스크를 켜고 끄는 함수 선언.

```bash
vi armcpu.c 
```

```c
#include "armcpu.h"

void enable_irq(void)
{
    __asm__ ("PUSH {r0, r1}");
    __asm__ ("MRS  r0, cpsr");
    __asm__ ("BIC  r1, r0, #0x80");
    __asm__ ("MSR  cpsr, r1");
    __asm__ ("POP {r0, r1}");
}

void enable_fiq(void)
{
    __asm__ ("PUSH {r0, r1}");
    __asm__ ("MRS  r0, cpsr");
    __asm__ ("BIC  r1, r0, #0x40");
    __asm__ ("MSR  cpsr, r1");
    __asm__ ("POP {r0, r1}");
}

void disable_irq(void)
{
    __asm__ ("PUSH {r0, r1}");
    __asm__ ("MRS  r0, cpsr");
    __asm__ ("ORR  r1, r0, #0x80");
    __asm__ ("MSR  cpsr, r1");
    __asm__ ("POP {r0, r1}");
}

void disable_fiq(void)
{
    __asm__ ("PUSH {r0, r1}");
    __asm__ ("MRS  r0, cpsr");
    __asm__ ("ORR  r1, r0, #0x40");
    __asm__ ("MSR  cpsr, r1");
    __asm__ ("POP {r0, r1}");
}
```

- ARM의 cspr의 IRQ/FIQ 마스크를 켜고 끄는 함수 구현.
- 0x80은 cspr의 IRQ 마스크 비트 위치인 7번 비트에 1이 있는 값.
- 0x40은 cspr의 FIQ 마스크 비트 위치인 6번 비트에 1이 있는 값.
- BIC 명령어를 사용하면 1이 있는 비트에 0을 쓴다. 마스크를 끄는 Enable 함수에 사용
- ORR 명령어를 사용하면 1이 있는 비트에 1을 쓴다. 마스크를 켜는 Disable 함수에 사용

>ARMCC는 컴파일러의 빌트인 변수로 cspr에 접근할 수 있지만, GCC는 그렇지 않아서 직접 어셈블리어를 사용해야 한다. 인라인 어셈블리어를 사용하여 작성한다. 인라인 어셈블리어를 사용할 경우 스택에 레지스터를 백업 및 복구하는 코드와 리턴 처리하는 코드를 컴파일러가 자동으로 만들어준다.

