---
title: "USART, I2C, SPI 통신 정리"
date: 2026-01-13 17:00:00 +0900
categories: [임베디드, 통신]
tags: [USART, I2C, SPI]
math: true
mermaid: true
---

이번에 RC Car 프로젝트를 진행하면서 블루투스 모듈(USART), LCD 디스플레이(I2C), RFID 리더(SPI)를 사용했다.  
하드웨어 핀을 연결하고 코드를 작성하면서 어느 정도 동작 원리를 이해했지만, 라이브러리에 의존한 부분도 있어 통신 이론을 완전히 이해하지 못했다.  
이 글에서는 프로젝트에서 사용한 세 가지 통신 방식(UART/USART, I2C, SPI)의 이론적인 내용을 정리한다.


## UART(Universal Asynchronous Receiver/Transmitter)
UART는 직렬 비동기 통신으로, 별도의 클럭 신호 없이 TX, RX 두 선만으로 통신이 가능하다.  
클럭 없이 동기화하기 위해 송수신자는 미리 약속된 **Baud rate(통신 속도)** 를 정확히 맞춰야 한다. 수신 측은 데이터 라인이 High에서 Low로 떨어지는 Start Bit를 감지하는 순간부터 보드레이트 주기에 맞춰 **샘플링**하며, 데이터는 일반적으로 LSB(Least Significant Bit) First 방식으로 전송된다.  
프레임은 보통 `Start bit + Data bits(5~9) + Parity(옵션) + Stop bit(1~2)` 구조를 가진다.

> 데이터 샘플링이란? 정해진 시간 간격으로 신호의 상태를 확인하여 그 값을 읽어내는 과정


## USART(Universal Synchronous/Asynchronous Receiver/Transmitter)
USART는 UART 비동기 기능에 동기식 기능이 추가된 상위 호환 인터페이스로, TX, RX, SCLK(Serial Clock) 세 개의 신호선을 사용해 데이터 전송 타이밍을 맞춘다.  
클럭 신호에 맞춰 데이터를 전송, 샘플링하기 때문에 비동기 UART보다 더 높은 속도와 낮은 타이밍 오차로 통신할 수 있다.


### STM32에서의 USART
사용 중인 STM32 보드(NUCLEO‑F411RE)는 USART를 지원하며, 몇 가지 주요 설정 포인트는 다음과 같다.

- **데이터 길이(Word Length)**  
  - `USART_CR1` 레지스터의 M 비트로 설정  
  - 8-bit: 일반적인 데이터 통신 (Start + 8 Data + Stop)  
  - 9-bit: 패리티 비트를 데이터와 함께 보내거나, 멀티‑드롭 환경에서 주소 비트로 활용할 때 사용


![usart bit](/assets/img/embedded/protocol/usart.png)

- **정지 비트(Stop Bit)**  
  - `USART_CR2` 레지스터의 STOP 비트로 설정  
  - 0.5 bit: 스마트카드 모드 수신 시 사용  
  - 1 bit: 기본 설정 (가장 많이 사용)  
  - 1.5 bits: 스마트카드 모드 송수신 시 사용  
  - 2 bits: 잡음 환경에서 프레임 간 간격을 늘려 안정성을 확보할 때 사용

![usart stop bit](/assets/img/embedded/protocol/usart_stop_bit.png)

## I2C(Inter-Integrated Circuit)  
I2C는 2선식 반이중 직렬 통신 방식으로, 데이터 라인 SDA와 클럭 라인 SCL 두 개의 신호선을 사용한다.  
두 라인은 풀업 저항을 통해 High로 유지되고, 모든 디바이스가 오픈드레인(또는 오픈콜렉터) 방식으로 Low를 끌어내리는 버스 구조를 가진다.  
I2C 인터페이스는 동작 역할에 따라 Target Transmitter, Target Receiver, Controller Transmitter, Controller Receiver의 네 가지 모드로 동작하며, 반이중 특성상 한 순간에는 송신 또는 수신 중 한 방향으로만 통신이 가능하다.  
또한 USART처럼 점대점(pin‑to‑pin) 연결이 아니라 Master–Slave(Controller–Target) 구조의 버스형 인터페이스이기 때문에, 하나의 I2C 마스터가 동일한 SDA/SCL 버스를 공유하는 여러 슬레이브 장치를 주소를 통해 개별적으로 제어할 수 있다.

![usart stop bit](/assets/img/embedded/protocol/i2c.png)

## SPI(Serial Peripheral Interface)
SPI는 동기식, 고속, 주로 풀‑듀플렉스(full‑duplex) 통신을 담당하는 직렬 인터페이스이다.  
공통 클럭(SCK)을 기준으로 마스터와 슬레이브가 동시에 데이터를 주고받을 수 있어, 같은 시간에 송신과 수신이 모두 가능한 것이 특징이다.  
기본 구성은 다음 네 개의 신호선을 사용한다.  
- **SCK (Serial Clock)**:  
  마스터가 생성하는 클럭 신호로, 데이터 샘플링 타이밍을 결정한다.
- **MOSI (Master Out, Slave In)**:  
  마스터 → 슬레이브 방향 데이터 라인.
- **MISO (Master In, Slave Out)**:  
  슬레이브 → 마스터 방향 데이터 라인.
- **NSS / SS (Slave Select)**:  
  활성화하고 싶은 슬레이브를 선택하는 칩 셀렉트 신호(Active Low).

SPI는 보통 하나의 마스터가 여러 슬레이브의 NSS를 각각 제어하는 구조로 사용하며, 필요에 따라 클럭 극성(CPOL)과 위상(CPHA)을 조합해 주변 장치와 타이밍을 맞춘다.  

![usart stop bit](/assets/img/embedded/protocol/spi.png)

## 언제 사용할까?

| 프로토콜 | 특징 | 적합한 용도 예시 |
|---------|------|------------------|
| UART/USART | 간단한 점대점 직렬 통신, 비교적 낮은 속도, 텍스트/디버그 출력에 적합 | PC 터미널 로그, BLE 모듈, GPS 모듈, 디버깅 출력 |
| I2C | 2선식 버스 구조, 다수 슬레이브 연결, 주소 기반 제어, 반이중 | 여러 센서, I2C LCD, EEPROM, PMIC 등 보드 내부 주변장치 |
| SPI | 고속, 풀‑듀플렉스, 선 수가 더 많음, 마스터가 NSS로 슬레이브 선택 | 고속 ADC/DAC, 외부 Flash, RFID 리더, 디스플레이, 고속 데이터 스트림 |


## 마무리
RC Car 프로젝트를 하면서는 “일단 동작하게 만드는 것”이 우선이라 세 통신 방식을 깊게 비교하지는 않았는데, I2C처럼 2선 버스로 여러 모듈을 같은 라인에 매달고 주소로 나눠서 제어할 수 있다는 점과, SPI처럼 공통 클럭과 데이터선을 공유하면서 슬레이브 선택 신호로 여러 디바이스를 관리하는 방식이 특히 인상적이었다.   
보드와 모듈 간 핀을 연결하면서 SPI의 핀 수가 많은 이유가 궁금했는데, 송신(MOSI)과 수신(MISO) 라인을 분리해서 클럭에 맞춰 동시에 데이터를 주고받는 풀‑듀플렉스 통신을 하기 위한 구조라고 이해하니 궁금증이 해결되었다.  
앞으로는 단순히 라이브러리를 가져다 쓰는 수준을 넘어서, 프로토콜 레벨에서 프레임 구조와 타이밍을 이해하고 선택할 수 있는 임베디드 개발자가 되는 것을 목표로 한다.
