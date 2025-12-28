---
title: "레지스터 직접 제어해보기(NUCLEO‑F411RE)"
date: 2025-12-28 20:00:00 +0900
categories: [임베디드, STM32]
tags: [HAL, LL, register]
math: true
mermaid: true
---


이전까지만 해도 STM32 에서 지원해주는 HAL 라이브러리를 활용하여 레지스터를 직접 제어하지 않아도 잘 구현할 수 있었다. 이런 기초적인 구현에서는 딱히 레지스터 제어를 할 필요는 없어보이지만 이후 데이터시트 읽는 방법이나 정확한 동작원리를 파악하기 위해 공부해본다.

# GPIO 란?
General-Purpose Input/Output의 약자로, 용도가 딱 정해져 있지 않고, 사용자가 소프트웨어로 설정하기에 따라 주변 장치와의 소통을 하는 등 다양한 용도로 쓸 수 있다.

## GPIO Register
GPIO 레지스터 중 많이 사용하는 5가지 레지스터를 정리했다.

### MODER
![MODER](/assets/img/embedded/setting//gpio_register/moder.png)

각 핀의 기본 동작 모드를 정하는 레지스터다.
핀 하나당 2비트씩 할당되어 있고, 값에 따라 아래와 같이 모드가 결정된다.  
- 입력(00): 외부 신호를 읽는 모드
- 일반 출력(01): 외부로 신호를 출력하는 모드
- Alternate Function(10): 해당 핀을 단순 GPIO가 아니라, 내부 주변장치에 연결된 전용 기능 핀으로 만드는 모드
- 아날로그 모드(11): 아날로그 입력 또는 출력 전용 회로로 맡기는 모드

### PUPDR
![PUPDR](/assets/img/embedded/setting//gpio_register/pupdr.png)

핀에 걸리는 내부 풀업/풀다운 저항을 설정하는 레지스터다. 버튼 처럼 floating 되면 안되는 곳에 설정하고 아래와 같은 모드가 있다.
- 없음(00)
- pull up(01)
- pull down(10)
- reserved(11): 동작이 정의되지 않았고 실사용하지 않는 않음

### IDR
![IDR](/assets/img/embedded/setting//gpio_register/idr.png)

Input Data Register로, 각 GPIO 핀의 현재 입력 값을 보여주는 레지스터다. 핀 하나당 1비트씩 매핑되어 있고, 값은 단순히 두 가지 상태만 가진다.
- 0: 해당 핀이 Low 가 입력된 상태
- 1: 해당 핀이 High 가 입력된 상태

이 비트를 읽어서 버튼이 눌렸는지, 센서 출력이 0/1인지 판단하는 데 사용한다.

### ODR
![IDR](/assets/img/embedded/setting//gpio_register/odr.png)

Output Data Register로, 각 GPIO 출력 핀에 내보내는 값을 저장하는 레지스터다. 핀 하나당 1비트씩 매핑되어 있고, 값은 다음 두 가지 상태를 가진다.

0 : 해당 핀에 Low 를 출력
1 : 해당 핀에 High 를 출력

핀을 출력 모드로 설정해 둔 상태에서 이 비트를 0/1로 바꾸면, LED ON/OFF나 릴레이 제어처럼 실제 핀 전압이 Low/High로 바뀌면서 외부 회로를 직접 제어할 수 있다.

### BSRR
![bsrr](/assets/img/embedded/setting//gpio_register/bsrr.png)

Bit Set/Reset Register로, 특정 핀을 한 번에 켜거나 끄는 전용 레지스터다. 32비트가 두 영역으로 나뉘어 있으며, 각 핀은 여기서도 1비트씩 매핑된다.

#### 하위 16비트(0~15): Set 영역(BS)
하위 16비트에 1을 쓰면 해당 핀의 출력이 High(1) 로 설정된다.

#### 상위 16비트(16~31): Reset 영역(BR)
상위 16비트에 1을 쓰면 해당 핀의 출력이 Low(0) 로 리셋된다.

ODR 전체를 읽고 수정해서 다시 쓰는 과정 없이, 원하는 핀 비트 하나만 바로 변경할 수 있기 때문에(원자성 보장), 인터럽트와 메인 루프에서 동시에 같은 포트를 건드는 상황에서도 더 안전하고 많이 사용된다.


## 레지스터 직접 제어
정리한 레지스터를 가지고 아래와 같은 간단한 시나리오를 구현할 것이다.
> 사용자 버튼(PC13)을 눌렀을 때 내부 LED(PA5)가 켜진다.

### 레지스터 설정
내부 LED를 제어하기 위해서는 GPIOA + 레지스터 offset 으로 지정해야하는데  
GPIOA는 `RM0383 Reference manual`의 `memory map`에서 GPIOA의 베이스 주소를 찾으면 되고  
offset은 위 레지스터 사진의 Address offset을 참고하거나 GPIO register map를 보고 offset을 찾을 수 있다.

![memory_map](/assets/img/embedded/setting//gpio_register/memory_map.png)

![register_map2](/assets/img/embedded/setting//gpio_register/register_map2.png)


> RCC 클럭 Enable GPIOA, GPIOC 같은 주변장치는 먼저 RCC에서 클럭을 켜야 레지스터가 정상 동작한다. `memory map`표를 보면 GPIOA/GPIOC가 AHB1 버스에 속해 있으므로, `RCC_AHB1ENR` 레지스터의 GPIOAEN(0), GPIOCEN(2)을 1로 설정하여 AHB1 클럭을 공급해 준다.
{: .prompt-tip }

![register_map2](/assets/img/embedded/setting//gpio_register/clock_enable_register.png)

#### 내부 LED에 설정할 레지스터
1. 포트: GPIOA(0x4002 0000), 핀: PA5
2. 출력: MODER 01  
3. 출력 제어: BSRR  
  - 켜기: `GPIOA_BSRR = 1U << 5;`
  - 끄기: `GPIOA_BSRR = 1U << (5 + 16);`  
  BSRR 상위 16비트를 키기위해 16을 더한다.

#### 내부 button에 설정할 레지스터
1. 포트: GPIOC(0x4002 0800), 핀: PC13
2. 입력포트로 설정: MODER 00
3. 풀업(보드 내부에 이미 외부 풀업이 달려 있음) PUPDR 00
4. 입력 읽기: IDR  
  사용자 버튼은 안 눌렀을 때 High(1), 눌렀을 때 Low(0)이다.

### 전체 코드
```c
#include <stdint.h>
#include "06_register_control.h"

// 베이스 주소
#define GPIOA_BASE 0x40020000UL
#define GPIOC_BASE 0x40020800UL
#define RCC_BASE 0x40023800UL

// RCC 레지스터
#define RCC_AHB1ENR  (*(volatile uint32_t *)(RCC_BASE  + 0x30))

// GPIOA 레지스터
#define GPIOA_MODER (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_OTYPER (*(volatile uint32_t *)(GPIOA_BASE + 0x04))
#define GPIOA_PUPDR (*(volatile uint32_t *)(GPIOA_BASE + 0x0C))
#define GPIOA_BSRR (*(volatile uint32_t *)(GPIOA_BASE + 0x18))

// GPIOC 레지스터
#define GPIOC_MODER (*(volatile uint32_t *)(GPIOC_BASE + 0x00))
#define GPIOC_PUPDR (*(volatile uint32_t *)(GPIOC_BASE + 0x0C))
#define GPIOC_IDR (*(volatile uint32_t *)(GPIOC_BASE + 0x10))

#define LD2_PIN 5
#define BTN_PIN 13


void gpio_register_run(void)
{
	// RCC에서 GPIOA, GPIOC 클럭 Enable 설정
	RCC_AHB1ENR |= (1U << 0);
	RCC_AHB1ENR |= (1U << 2);

	// 내부 LED 설정
	GPIOA_MODER &= ~(0x3U << (LD2_PIN * 2));
	GPIOA_MODER |= (0x1U << (LD2_PIN * 2));

	// PC13 내부 버튼 설정
	GPIOC_MODER &= ~(0x3U << (BTN_PIN * 2));
	GPIOC_PUPDR &= ~(0x3U << (BTN_PIN * 2));

	while(1)
	{
		uint32_t btn = (GPIOC_IDR >> BTN_PIN) & 0x1;

		if (btn == 0) {
			GPIOA_BSRR = (1U << LD2_PIN);
		}
		else {
			GPIOA_BSRR = (1U << (LD2_PIN + 16));
		}
	}
}
```

#### 코드 해석
코드를 작성하면서 궁금했던 점을 몇개 작성하자면

```c
#define GPIOC_IDR (*(volatile uint32_t *)(GPIOC_BASE + 0x10))
```
처음에는 *(volatile uint32_t *)를 굳이 이렇게까지 써야 하나 싶었는데, 정리해 보면 다음처럼 이해할 수 있었다.
- GPIOC_BASE + 0x10 자체는 그냥 정수 값(주소 숫자)일 뿐이다.
- (uint32_t *)는 “이 숫자를 32비트 정수가 있는 메모리 주소로 해석해라”라는 의미의 포인터 캐스팅이다.
- 앞의 *는 그 포인터가 가리키는 실제 32비트 값을 읽고/쓰겠다는 뜻이다.
- volatile은 이 값이 하드웨어에 의해 언제든 바뀔 수 있으니, 컴파일러가 캐시하거나 최적화로 생략하지 말고 항상 메모리에서 직접 읽고 쓰도록 강제하는 키워드다.  

즉, 이 한 줄은 C에게 “여기는 32비트짜리 GPIOC_IDR 레지스터가 있는 주소고, 값은 외부(하드웨어)가 수시로 바꿀 수 있으니까 매번 메모리에서 읽어라”라고 설명해 주는 코드라고 볼 수 있다.

2번째로 메인 루프의 설정 코드에서 처음 헷갈렸던 부분은 이 부분이다.
```c
GPIOA_MODER &= ~(0x3U << (LD2_PIN * 2));
```
LD2_PIN * 2처럼 핀 번호에 2를 곱하는 이유는, MODER 같은 레지스터가 핀 하나당 2비트씩을 모드 설정에 쓰는 구조이기 때문이다.  
레퍼런스 매뉴얼의 MODER 표를 보면
- MODER0[1:0] → bit 1:0
- MODER1[1:0] → bit 3:2  
이걸 깨닫고 나니, 핀당 2비트 쓰는 레지스터(MODER, PUPDR, OSPEEDR, AFR)에는 핀번호 * 2를 쓰고,  
1비트만 쓰는 레지스터(OTYPER, IDR, ODR, BSRR)에는 그냥 핀번호만 shift 하는 패턴으로 정리할 수 있었다.

## 같은 동작을 HAL로 구현하기
위 레지스터 코드를 “동작은 동일하게, 표현만 HAL 함수로 바꾼 버전”이라고 보면 된다.
```c
#include "main.h"

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();      // GPIO 초기화만 사용

  while (1)
  {
    // PC13 버튼이 눌렸는지 확인 (눌리면 0)
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
    {
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);   // LD2 ON
    }
    else
    {
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); // LD2 OFF
    }
  }
}
```

## 마치며
간단한 예제지만 레지스터를 직접 만져 보면서 GPIO가 실제로 어떻게 동작하는지 몸으로 익혀 볼 수 있었다. HAL로 같은 기능을 만들 때보다 코드 양도 많고, 매뉴얼을 보느라 시간도 최소 몇 배는 더 들었지만, 그 덕분에 이제 HAL 함수가 내부에서 어떤 비트를 어떻게 건드리는지 어느 정도 그림이 그려진다. 사실 대부분 HAL이나 LL을 쓰겠지만, 이번 경험처럼 한 번쯤 레지스터를 직접 제어해 본 덕에 데이터시트와 레퍼런스 매뉴얼을 읽는 감각을 길렀다는 점에 의미를 두고 싶다.