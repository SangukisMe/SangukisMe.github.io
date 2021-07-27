---
title: "임베디드 OS 개발 프로젝트 6장 :: INTERRUPT(2)"
excerpt: "UART 입력 및 인터럽트 연결"

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

# 6.2 UART 입력과 인터럽트 연결

```bash
vi Uart.c
```

```c
#include "stdint.h"
#include "Uart.h"
#include "HalUart.h"
#include "HalInterrupt.h"

extern volatile PL011_t* Uart;

static void interrupt_handler(void);

void Hal_uart_init(void)
{
    // Enable UART
    Uart->uartcr.bits.UARTEN = 0;
    Uart->uartcr.bits.TXE = 1;
    Uart->uartcr.bits.RXE = 1;
    Uart->uartcr.bits.UARTEN = 1;

    // Enable input interrupt
    Uart->uartimsc.bits.RXIM = 1;

    // Register UART interrupt handler
    Hal_interrupt_enable(UART_INTERRUPT0);
    Hal_interrupt_register_handler(interrupt_handler, UART_INTERRUPT0);
}

//… 후략 …//
```

- UART input 인터럽트를 활성화한다.
- UART IRQ ID인 44번 인터럽트를 활성화하고, interrupt_handler() 함수를 UART 인터럽트 핸들러로 등록한다.

```c
// Uart.c

static void interrupt_handler(void)
{
    uint8_t ch = Hal_uart_get_char();
    Hal_uart_put_char(ch);
}
```

- UART 인터럽트 핸들러로 등록된 함수이다. 인터럽트를 처리하는 코드를 이 함수에 작성한다.
- UART 입력이 발생하면 코어가 자동으로 이 함수를 실행한다.

```bash
vi Main.c
```

```c
#include "stdint.h"
#include "HalUart.h"
#include "stdio.h"
#include "HalInterrupt.h"

static void Hw_init(void);
static void Printf_test(void);
	
void main(void)
{
	Hw_init();

	uint32_t i = 100;
	while(i--)
	{
		Hal_uart_put_char('N');
	}
	Hal_uart_put_char('\n');

	putstr("Hello World!\n");
	
	Printf_test();

	while(1);

}

static void Hw_init(void)
{
	Hal_interrupt_init();
	Hal_uart_init();
}

//… 후략 …//
```

- 인터럽트 동작을 확인하기 위해 main() 함수에서 while(1)문을 이용해 무한 루프를 만들어준다.
- Hal_uart_init() 함수 내부에서 인터럽트 관련 함수를 호출하므로 그 전에 인터럽트 컨트롤러를 먼저 초기화해 놔야 정상적으로 동작한다.

<br/>

# 6.3 IRQ 익셉션 벡터 연결

```bash
vi Handler.c
```

```c
#include "stdbool.h"
#include "stdint.h"
#include "HalInterrupt.h"

__attribute__ ((interrupt ("IRQ"))) void Irq_Handler(void)
{
	Hal_interrupt_run_handler();
}

__attribute__ ((interrupt ("FIQ"))) void Fiq_Handler(void)
{
	while(true);
}
```

- IRQ와 FIQ의 익셉션 핸들러 구현. FIQ 익셉션 핸들러는 더미(dummy)이다.
- ```__attribute__``` : GCC의 컴파일러 확장 기능을 사용하겠다는 지시어.
- ```__attribute__ ((Interrupt ("IRQ")))``` : ARM용 GCC의 전용 확장 기능. IRQ의 핸들러에 진입하는 코드와 나가는 코드를 컴파일러가 자동으로 만들어 준다. 

>바로 이 익셉션 핸들러 함수로 진입하는 과정까지가 하드웨어적으로 처리되는 부분이다. 이 이후부터는 소프트웨어적인 작업이므로 익셉션 핸들러에 진입하기 전에 리턴 주소를 저장할 필요가 있다. 이때 ```__attribute__ ((Interrupt ("INTERRUPT")))``` 지시어를 사용하면 리턴 주소를 저장하는 작업과 리턴 주소를 이용해 나가는 작업을 자동으로 만들어준다. 

```bash
vi Entry.S
```

```c
#include "ARMv7AR.h"
#include "MemoryMap.h"

.text
	.code 32

	.global vector_start
	.global vector_end

	vector_start:
	 LDR PC, reset_handler_addr
	 LDR PC, undef_handler_addr
	 LDR PC, svc_handler_addr
	 LDR PC, pftch_abt_handler_addr
	 LDR PC, data_abt_handler_addr
	 B .
	 LDR PC, irq_handler_addr
	 LDR PC, fiq_handler_addr

	 reset_handler_addr:		.word reset_handler
	 undef_handler_addr:		.word dummy_handler
	 svc_handler_addr:		.word dummy_handler
	 pftch_abt_handler_addr:	.word dummy_handler
	 data_abt_handler_addr:		.word dummy_handler
	 irq_handler_addr:		.word Irq_Handler
	 fiq_handler_addr:		.word Fiq_Handler	
	vector_end:

/… 후략 …/
```

- 익셉션 벡터 테이블에서 익셉션 핸들러 함수로 연결.
- IRQ 익셉션 : Irq_Handler 로 연결.
- FIQ 익셉션 : Fiq_Handler 로 연결.

```bash
make run
```

- 정상적으로 동작한다면 키보드로 입력한 문장이 계속해서 출력된다.

