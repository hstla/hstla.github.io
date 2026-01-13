---
title: "STM32로 신호등 타이머 만들기(TM1637)"
date: 2026-01-05 12:00:00 +0900
categories: [임베디드, STM32]
tags: [GPIO, TM1637, 7-segment]
math: true
mermaid: true
---

# 신호등 타이머 구현
지난 블로그에서는 LED와 타이머를 활용해서 신호등을 만들었다. [신호등 포스팅](https://hstla.github.io/posts/traffic_light/)  
이번에는 4-digit-display 모듈을 추가해서 신호등 타이머 기능을 추가해볼 것이다.

## 사용한 모듈
구매한 모듈은 TM1637로 TM1637는 4자리 7세그먼트와 드라이버 IC가 하나로 묶인 모듈이라, 적은 핀으로 숫자를 쉽게 표시할 수 있다. [디바이스 마트 TM1637 링크](https://www.devicemart.co.kr/goods/view?no=1326952)  

### TM1637 회로
TM1637 모듈은 4개의 핀(CLK, DIO, VCC, GND)으로 구동된다. 아래 회로를 보면, 왼쪽의 R1, R2에 각각 풀업 저항이 연결되어 있어 CLK와 DIO 라인을 VCC 쪽으로 끌어올려 준다. 이 구조 덕분에 MCU에서는 두 라인을 Open‑Drain 모드로 설정한 뒤, 필요한 순간에만 0으로 당겨서 신호를 제어할 수 있다.

![tm1637회로사진](/assets/img/embedded/circuit/tm1637-display-module-circuit.png)

## 구현 과정
내가 구입한 모듈은 `88:88` 형태로 되어 있어 시계처럼 `MM:ss` 형태로 표현하기 좋게 되어 있는데 내가 원하는 표현방식은 `ss.tt` 이다. 하는 수 없이 앞뒤를 제외하고 가운데 두 자리만 사용하여 `_s:t_`를 구현할 것이다.

### pin out 설정
TM1637 모듈은 CLK, DIO, VCC(+5V), GND 총 4개의 핀만으로 동작한다. 이번 프로젝트에서는 Nucleo 보드의 CN7 헤더에서 GND와 +5V를 전원으로 사용하고, PC10과 PC11을 각각 CLK와 DIO로 할당했다.  
PC10, PC11은 초기 출력 레벨을 High로 두고, GPIO 모드는 Output Open‑Drain, 속도는 High로 설정했다. TM1637 회로를 보면 두 라인에 풀업 저항이 걸려 있기 때문에, Open‑Drain에서는 High일 때 MCU가 핀을 떠나 두고(Hi‑Z 상태, 풀업이 라인을 VCC로 끌어올림), Low일 때만 MCU가 직접 0을 출력해 GND로 당겨 주는 방식으로 안전하게 버스를 제어할 수 있다.

![tm1637pin_out_setting](/assets/img/embedded/led/tm1637_pinout_setting.png)

이렇게 설정하고 GENERATE CODE 버튼을 누르면 아래와 같이 GPIO 설정이 추가된다.

```c
  HAL_GPIO_WritePin(GPIOC, TM1637_CLK_Pin|TM1637_DIO_Pin, GPIO_PIN_SET);
  ...

  GPIO_InitStruct.Pin = TM1637_CLK_Pin|TM1637_DIO_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
```

### 코딩
TM1637 구현을 위해 아래 깃허브의 TM1637 라이브러리에서 tm1637.c, tm1637.h, tm1637_config.h 파일을 가져와 프로젝트에 포함했다.

[TM1637 라이브러리](https://github.com/nimaltd/tm1637?tab=readme-ov-file)

기본적인 기능 확인을 위해 기본적인 코드로 테스트를 진행했다.

```c
#include "tm1637.h"
...
tm1637_t seg =
  {
      .seg_cnt  = 4,
      .gpio_clk = TM1637_CLK_GPIO_Port,
      .gpio_dat = TM1637_DIO_GPIO_Port,
      .pin_clk  = TM1637_CLK_Pin,
      .pin_dat = TM1637_DIO_Pin,
  };

  tm1637_init(&seg); // 4-digital display 초기화
  tm1637_brightness(&seg, 1);  // display 밝기 설정
  tm1637_str(&seg, "1234");
```
참고로 화면 밝기는 0부터 8까지 설정할 수 있는데, 실사용 시에는 너무 눈부셔서 최저 값인 1로 고정해 사용했다.


#### 주간 모드
주간 모드에서는 GREEN(3s) -> YELLOW(1s) -> RED(2s) 를 LED와 같이 계속 반복해서 보여줘야 한다.  
먼저 남은 시간을 `_s:t_` 표현으로 표시해주는 함수를 만들었다.

```c
static void display_countdown(uint32_t remaining_ms)
{
    char buf[4];
    uint32_t deciseconds = remaining_ms / 100;
    uint32_t seconds     = deciseconds / 10;
    uint32_t deci        = deciseconds % 10;

    sprintf(buf, "%02lu.%1lu", seconds, deci);

    if (buf[0] == '0')
    {
    	buf[0] = ' ';
    }
    tm1637_str(&seg, buf);
}
```
함수의 입력이 만약 2900 으로 들어온다면 deciseconds = 29, seconds = 2, deci = 9 가 된다. 이를 `0s:t_`로 표현하기 위해 `"%02lu.%1lu"` 표현식을 사용했는데 이는 앞 2자리 0패딩된 unsigned long, 뒤쪽은 1자리 unsigned long을 소수점으로 구분해 출력한다.  
지금 코드에서는 s가 2자리가 넘어가지 않기 때문에 앞자리를 비워 한자리로 보이게 하게 했다.
```c
if (buf[0] == '0')
    {
    	buf[0] = ' ';
    }
```

#### 야간 모드
야간 모드에서는 원래 YELLOW LED만 1초 간격으로 총 6번 점멸했지만, 여기에 TM1637 디스플레이도 함께 점멸하도록 동작을 확장했다.  
아래 코드는 야간 모드에서 디스플레이를 제어하는 함수다.
​
```c
...
if (night_digit_display)
    {
        night_digit_display = false;
        tm1637_clear(&seg);
    }
    else
    {
        night_digit_display = true;
        tm1637_str(&seg, "88.88");
    }
```

기존 코드에서 아래 코드만 추가한 것이다. 사용자 버튼이 눌러지면 곧바로 `night_digit_display = true`로 설정되고 `tm1637_str(&seg, "88.88");` 코드로 디스플레이가 켜진다. 이후 6번 점멸하고 다시 주간 모드로 돌아간다.

#### 전체 코드
[신호등 메인 코드 URL](https://github.com/hstla/stm32_led/blob/main/Core/Src/07_traffic_light.c)

### 구현 결과

<div style="text-align: center;">
  <video autoplay loop muted playsinline width="400">
    <source src="/assets/img/embedded/led/traffic_display.MOV" type="video/mp4">
  </video>
</div>

## 마무리
이전 글에서 구현했던 LED만 있는 신호등보다 훨씬 직관적으로 남은 시간을 표시할 수 있었고, 주간/야간 모드 전환도 눈으로 바로 확인할 수 있었다. 다만 0.1초 단위로 숫자가 계속 바뀌다 보니 장시간 보면 눈이 조금 피로해지긴 했지만, 타이머 동작을 검증하는 용도로는 만족스러웠다.