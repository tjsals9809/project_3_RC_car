# 🚗 Project_3 Autonomous RC car 🚗

## 📌 1. Project Summary (프로젝트 요약)

STM32(MCU)를 활용하여 블루투스를 통한 수동조종(Manual) 및 자율주행(Auto) 시스템 제작

<br>

## ✨ 2. Key Features (주요 기능)

### 🕹️ 2.1 Manual Mode (수동 제어)
스마트폰 어플을 이용한 수동 주행 제어
전진 / 후진 / 좌회전 / 우회전 / 정지 동작

### 🤖 2.2 Auto Mode (자율주행)
전방, 좌측, 우측 3개의 초음파 센서를 이용해 거리를 측정하여 장애물 회피
상태 머신 기반 자율주행 로직 구현
데이터를 이중으로 비교하여 회전 중에도 재판단
코너에 진입했을시 전면과의 거리가 너무 가까우면 넓은 방향으로 후진

### 2.3 Control Command (제어 명령)
| 명령  | 동작              |
| --- | --------------- |
| `F` | 전진              |
| `B` | 후진              |
| `L` | 좌회전             |
| `R` | 우회전             |
| `S` | 정지              |
| `A` | 자율주행 모드 시작      |
| `M` | 자율주행 모드 종료 및 정지 |
<br>

## ⚙️ 3. Tech Stack (기술 스택)

### 3.1 Language (사용 언어)

![C](https://img.shields.io/badge/C-00599C?style=for-the-badge\&logo=c\&logoColor=white)

### 3.2 Development Tool (개발 환경)

![STM32CubeIDE](https://img.shields.io/badge/STM32CubeIDE-03234B?style=for-the-badge\&logo=stmicroelectronics\&logoColor=white)
![STM32CubeMX](https://img.shields.io/badge/STM32CubeMX-03234B?style=for-the-badge\&logo=stmicroelectronics\&logoColor=white)

### 3.3 Hardware (하드웨어)

| 구분            | 내용                           |
| ------------- | ---------------------------- |
| MCU           | STM32F411RE                  |
| Motor         | DC Motor 2개                  |
| Sensor        | Ultrasonic Sensor 3개         |
| Communication | Bluetooth Module             |
| Debug         | USART2 Serial Output         |
| Motor Control | PWM + GPIO Direction Control |
| Driver        | STM32 HAL Driver             |

### 3.4 Peripheral (주변장치)

| Peripheral    | 사용 목적                     |
| ------------- | ------------------------- |
| TIM3 CH1      | 전방 초음파 센서 Echo 입력 캡처      |
| TIM3 CH2      | 좌측 초음파 센서 Echo 입력 캡처      |
| TIM3 CH3      | 우측 초음파 센서 Echo 입력 캡처      |
| TIM10 PWM CH1 | Motor A 속도 제어             |
| TIM11 PWM CH1 | Motor B 속도 제어             |
| USART1        | 블루투스 모듈 수신                |
| USART2        | 디버깅용 시리얼 출력               |
| GPIO          | 모터 방향 제어 및 초음파 Trigger 출력 |

<br>

## 🧩 4. Project Structure (프로젝트 구조)

```bash
project_RC/
├── Core/
│   ├── Inc/                         # 각 소스 모듈에 대응하는 헤더 파일 (.h)
│   └── Src/                         # 프로젝트 핵심 로직 구현부 (.c)
│       ├── main.c                   # 하드웨어 초기화 및 자율주행 상태 머신 제어
│       ├── motor.c                  # Motor A/B PWM 속도 및 방향 제어
│       ├── bluetooth.c              # 블루투스 명령 수신 및 주행 명령 처리
│       ├── ultrasonic.c             # 초음파 센서 Trigger 신호 발생
│       ├── delay.c                  # us 단위 지연 처리
│       ├── tim.c                    # PWM 및 Input Capture 타이머 설정
│       ├── usart.c                  # 블루투스/디버깅 UART 설정
│       └── gpio.c                   # 모터 방향 핀 및 초음파 Trigger 핀 설정
│
├── Drivers/                         # STM32 HAL 드라이버 및 CMSIS 라이브러리
├── project_RC.ioc                   # STM32CubeMX 하드웨어 구성 및 핀 설정 파일
├── STM32F411RETX_FLASH.ld           # Flash 메모리 링커 스크립트
├── STM32F411RETX_RAM.ld             # RAM 메모리 링커 스크립트
└── README.md                        # 프로젝트 전체 가이드 문서
```

<br>

## 🎬 5. Final Product & Demonstration (완성품 및 시연)
<br>

### 5.1 Final Product (완성품)
<br>
<img src="./images/rc_car.jpg" width="400">

<br>

### 5.2 Demonstration (시연 영상)
<br>

<a href="https://www.youtube.com/watch?v=y_h5x1xk92Q">
  <img src="./images/youtube.png" width="500">
</a>

### *이미지를 클릭하면 영상으로 이동합니다.*

<br><br>




## 🕹️ 6. Autonomous Driving Logic (자율주행 로직)

자율주행 모드는 전방, 좌측, 우측 초음파 센서의 거리값을 기준으로 상태를 전환합니다.

```c
typedef enum {
  STATE_FORWARD = 0,
  STATE_BACKWARD,
  STATE_TURN_LEFT,
  STATE_TURN_RIGHT
} CarState;
```

| 상태                 | 동작                          |
| ------------------ | --------------------------- |
| `STATE_FORWARD`    | 기본 전진 상태                    |
| `STATE_BACKWARD`   | 전방 장애물 감지 시 후진              |
| `STATE_TURN_LEFT`  | 우측 장애물 감지 또는 좌측 공간 확보 시 좌회전 |
| `STATE_TURN_RIGHT` | 좌측 장애물 감지 또는 우측 공간 확보 시 우회전 |

### 주요 기준값

```c
#define DANGER_DIST     13
#define RECOVER_DIST    25
#define BASE_SPEED      300
#define TURN_SPEED      300
#define BACK_SPEED      200
#define ACTION_TIME     300
```

* `DANGER_DIST` 이하로 장애물이 감지되면 회피 동작 수행
* `RECOVER_DIST` 이상으로 공간이 확보되면 전진 상태로 복귀
* `ACTION_TIME`을 이용해 후진 및 회전 동작 시간을 제한
<br>
<br>

## 🎯 7. Troubleshooting (문제 해결 기록)

## 7.1 초음파 센서값이 0cm로 튀면서 장애물 판단이 불안정한 문제

#### 🔍 문제 상황

초음파 센서가 순간적으로 `0cm` 또는 매우 작은 값으로 측정되면서 실제로 장애물이 없는데도 장애물로 판단하는 문제가 발생했습니다.

#### ❓ 원인 분석

Echo 신호가 제대로 들어오지 않거나 센서 측정이 순간적으로 실패하면 거리값이 `0cm`에 가깝게 계산될 수 있습니다.

#### ❗ 해결 방법

`3cm` 이하의 비정상적인 값은 실제 장애물로 판단하지 않고, 임시로 먼 거리값인 `999`로 처리

```c
uint16_t fd = (Front_distance <= 3) ? 999 : Front_distance;
uint16_t ld = (Left_distance <= 3) ? 999 : Left_distance;
uint16_t rd = (Right_distance <= 3) ? 999 : Right_distance;
```

이를 통해 센서 튐으로 인해 불필요하게 후진하거나 회전하는 문제를 줄였습니다.

---

## 7.3 자율주행 중 동작 전환이 복잡해지는 문제

#### 🔍 문제 상황

전진, 후진, 좌회전, 우회전을 단순 조건문으로만 처리하면 장애물 위치에 따라 동작이 빠르게 바뀌어 주행이 불안정해질 수 있었습니다.

#### ❓ 원인 분석

RC카는 현재 어떤 동작을 수행 중인지에 따라 다음 동작이 달라져야 합니다. 하지만 상태 구분 없이 조건문만 사용할 경우, 전방 장애물 감지와 좌우 회피 조건이 한 루프 안에서 섞여 동작 흐름이 꼬일 수 있습니다.

#### ❗ 해결 방법

자율주행 로직을 상태 머신 형태로 구성했습니다.

```c
switch(currentState)
{
  case STATE_FORWARD:
    break;

  case STATE_BACKWARD:
    break;

  case STATE_TURN_LEFT:
    break;

  case STATE_TURN_RIGHT:
    break;
}
```

## 7.4 블루투스 명령 수신이 한 번만 동작하는 문제

#### 🔍 문제 상황

UART 인터럽트 방식으로 블루투스 데이터를 받을 때, 수신 인터럽트를 다시 등록하지 않으면 다음 명령을 받을 수 없는 문제가 발생할 수 있습니다.

#### ❓ 원인 분석

`HAL_UART_Receive_IT()`는 1바이트 수신이 완료되면 콜백 함수가 실행되고, 이후 다음 수신을 위해 다시 호출해 주어야 합니다.

#### ❗ 해결 방법

블루투스 수신 콜백 함수 마지막에서 다시 수신 인터럽트를 등록했습니다.

```c
HAL_UART_Receive_IT(&huart1, (uint8_t *)&Rx_data, 1);
```

이를 통해 `F`, `B`, `L`, `R`, `S`, `A`, `M` 명령을 연속적으로 받을 수 있도록 구현했습니다.

## 🔧 8. 개선 사항

* 주행 속도값을 고정값이 아니라 단계별 조절 가능하도록 확장
* 배터리 전압 측정 기능을 추가하여 저전압 상태를 사용자에게 알림
* 카메라 모듈을 추가하여 원격 조종할 수 있는 구조로 확장
