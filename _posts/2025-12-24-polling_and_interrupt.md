---
title: "인터럽트와 폴링의 차이 알아보기(NUCLEO‑F411RE)"
date: 2025-12-24 20:00:00 +0900
categories: [임베디드, STM32]
tags: [인터럽트, 폴링]
math: true
mermaid: true
---
# 폴링(polling) 과 인터럽트(interrupt)
폴링과 인터럽트는 둘 다 장치에서 발생하는 이벤트를 어떻게 감지하고 처리할지를 결정하는 방식이다.  
폴링은 장치의 상태를 주기적으로 계속 확인하는 방식이고, 인터럽트는 장치가 이벤트가 발생한 순간 CPU에 신호를 보내도록 해서 그때 바로 감지하고 처리하는 방식이다. 이 두가지 방식을 시나리오를 구현하면서 차이점을 알아보았다.

## 예제 시나리오
메인 기능은 내부 LED(PA5)가 2초간 켜지고 1초간 꺼지는 동작을 반복하는 것이다.  
장치에서 발생하는 이벤트로는 사용자 버튼(PC13)을 누르면 외부 LED(PC9)가 2초간 켜지고 다시 메인 기능으로 로직이 넘어간다.

### 폴링 및 인터럽트 예상 시나리오
- **폴링 방식**: 메인 루프가 순차적으로 실행되므로, `HAL_Delay()` 구간에서 버튼을 눌러도 즉시 반응하지 않고 현재 동작이 끝난 후에야 감지됨
- **인터럽트 방식**: 버튼이 눌리는 순간 하드웨어 신호가 발생하여 CPU가 즉시 인터럽트 핸들러를 실행하여 이벤트 처리

## 폴링방식으로 버튼 처리하기
폴링 방식은 메인 루프에서 계속해서 버튼 상태를 확인하는 방식이다.

```c
#include <stdbool.h>
#include "04_polling.h"
#include "main.h"

void led_polling_run()
{
	while(1) {
		if(HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
		{
			HAL_GPIO_WritePin(EXT_GPIO_Port, EXT_LED_Pin, GPIO_PIN_SET);
			HAL_Delay(2000);
			HAL_GPIO_WritePin(EXT_GPIO_Port, EXT_LED_Pin, GPIO_PIN_RESET);
		}

		HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
		HAL_Delay(2000);
		HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
		HAL_Delay(1000);
	}
}
```

### 폴링의 장단점
**장점**
인터럽트 설정을 따로 할 필요가 없어, 구현이 단순하고 직관적이다.  
이벤트 발생 빈도가 낮고 타이밍이 중요하지 않은 경우 적합하다.

**단점**
CPU가 계속 이벤트를 확인해야 하므로 비효율적이며, 응답 속도가 느리고 실시간성이 떨어진다. 특히 `Delay`구간에서는 이벤트를 놓칠 수 있어 사용자가 타이밍에 맞춰 버튼을 눌러야 이벤트가 동작한다.

## 인터럽트방식으로 버튼 처리하기
인터럽트 방식은 하드웨어가 이벤트를 감지하면 CPU에 신호를 보내 즉시 처리하는 방식이다.

```c
#include 
#include "00_timer2.h"
#include "05_interrupt.h"
#include "main.h"

static volatile bool ext_led_active = false;
static volatile uint32_t ext_led_end = 0;

static void ext_led_task(void);
static void internal_led_task(void);

void led_interrupt_run(void)
{
	// 타이머 콜백 등록
	tim2_register_callback(internal_led_task);
	tim2_register_callback(ext_led_task);
}

// 버튼 인터럽트 콜백 (버튼 눌림 시 즉시 실행)
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if (GPIO_Pin == B1_Pin)
	{
		// 내부 LED 즉시 끄기
		HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
		// 외부 LED 켜기
		HAL_GPIO_WritePin(EXT_GPIO_Port, EXT_LED_Pin, GPIO_PIN_SET);

		ext_led_active = true;
		ext_led_end = get_tim2_ms() + 2000;
	}
}

// 외부 LED 관리 태스크
static void ext_led_task(void)
{
	if (!ext_led_active) {
		return;
	}

	if ((int32_t)(get_tim2_ms() - ext_led_end) >= 0)
	{
		HAL_GPIO_WritePin(EXT_GPIO_Port, EXT_LED_Pin, GPIO_PIN_RESET);
		ext_led_active = false;
	}
}

// 내부 LED 깜빡임 태스크
static void internal_led_task(void)
{
	static uint32_t next_toggle = 0;
	static bool led_on = false;

	// 외부 LED 동작 중에는 내부 LED 중지
	if (ext_led_active) {
		return;
	}

	if ((int32_t)(get_tim2_ms() - next_toggle) >= 0)
	{
		led_on = !led_on;
		HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, led_on ? GPIO_PIN_SET : GPIO_PIN_RESET);

		next_toggle = get_tim2_ms() + (led_on ? 2000 : 1000);
	}
}
```
### 인터럽트 방식의 동작
1. TIM2 타이머가 1ms마다 등록된 함수를 호출
2. `internal_led_task()`가 타이밍을 계산하여 내부 LED 제어
3. 버튼을 누르면 즉시 `HAL_GPIO_EXTI_Callback()` 실행
4. 외부 LED가 2초동안 켜지고 내부 LED는 일시 정지
5. 2초 후 `ext_led_task()`가 외부 LED를 끄고 내부 LED 동작

### 인터럽트의 장단점
**장점**  
이벤트 발생 시 즉시 반응하여 실시간성이 우수하다.  
CPU가 다른 작업 중에도 중요한 이벤트를 놓치지 않고 처리할 수 있다.  
이벤트가 없을 때는 CPU가 다른 작업을 수생할 수 있어 효율적이다.  

**단점**  
설정이 폴링방식에 비해 복잡하다. 특히 STM32CubeMX에서 인터럽트 설정하지 않으면 필요한 NVIC 설정 코드나 인터럽트 핸들러가 생성되지 않아 추가 작어비 필요하다.  
인터럽트가 너무 자주 발생하면 컨텍스트 스위칭 오버헤드가 누적되어 오히려 성능이 저하될 수 있다.  
여러 인터럽트를 관리할 때 우선순위 설정과 디버깅이 복잡해진다.  

> 이번 예제에서는 TIM2 타이머 인터럽트에 여러 콜백을 등록하는 방식을 사용했는데, 로직이 더 복잡해지면 각 콜백 간의 상호작용이나 타이밍 이슈 등을 디벙깅하기 어려워질 수 있다.

## 차이점 정리
| 구분 | 폴링 | 인터럽트 |
|------|------|----------|
| **응답 속도** | 느림 (루프 주기에 의존) | 빠름 (즉시 반응) |
| **CPU 효율** | 낮음 (계속 확인) | 높음 (이벤트 발생 시만) |
| **구현 난이도** | 쉬움 | 보통 |
| **실시간성** | 낮음 | 높음 |
| **적합한 경우** | 이벤트 빈도 낮음, 타이밍 덜 중요 | 빠른 응답 필요, 실시간 처리 |


## 언제 어떤 방식을 선택해야 하나?
**폴링을 선택하는 경우**  
- 이벤트 발생 빈도가 낮고 타이밍이 중요하지 않을 때
- 간단한 시스템이나 프로토타입
- 인터럽트 설정이 복잡하거나 불가능한 환경

**인터럽트를 선택하는 경우**  
- 빠른 응답이 필요할 때 (예: 긴급 정지 버튼)
- 이벤트 발생 시점을 정확히 감지해야 할 때
- CPU 효율을 높여야 하는 배터리 구동 기기
- 여러 작업을 동시에 처리해야 할 때


### 마치며
이번 실습 전에는 "인터럽트가 무조건 좋은 거 아닌가?"라는 막연한 생각을 했었다. 하지만 직접 구현해보니 각 방식마다 장단점이 명확했고, 상황에 맞게 선택하거나 조합해야 한다는 것을 깨달았다.

특히 인터럽트에서는 간단한 플래그 설정만 하고, 복잡한 로직은 메인 루프에서 처리하는 패턴이 가장 안정적이고 디버깅하기 쉬웠다. 인터럽트 핸들러는 가능한 한 짧고 빠르게 유지하고, 무거운 작업은 메인 루프로 넘기는 것이 좋은 설계라는 것을 배웠다.

앞으로 임베디드 프로젝트를 진행할 때 이 두 방식을 적재적소에 활용할 수 있을 것 같다.