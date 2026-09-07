# STM32 HAL 경험에서 Android HAL 이해로 확장한 정리

## 1. STM32

STM32를 사용할 때 접했던 HAL은 다음과 같은 의미로 이해하고 있었다.

- HAL = Hardware Abstraction Layer
- 레지스터를 직접 조작하지 않아도 주변장치를 쉽게 제어할 수 있도록 ST가 제공하는 라이브러리
- GPIO, UART, I2C, SPI, Timer 등을 비교적 일관된 API로 사용할 수 있게 해주는 추상화 계층

예를 들어 STM32에서는 다음과 같은 구조로 이해할 수 있다.

```text
Application Code
    ↓
STM32 HAL API
    ↓
Peripheral Register
    ↓
MCU Hardware
```

```c
HAL_GPIO_WritePin(...);
HAL_UART_Transmit(...);
```

이때의 HAL은 기본적으로 개발 편의를 위한 추상화에 가깝게 이해하고 있었다.

즉, STM32에서는 필요하면 HAL을 우회하고 직접 레지스터에 접근할 수 있었다.

---

## 2. Android

프로젝트를 진행하면서,
Android의 센서 구조를 조사하면서 다음과 같은 계층을 보게 되었다.

```text
Android Application
    ↓
Android Framework API
    ↓
System Service
    ↓
Android HAL
    ↓
Vendor Implementation
    ↓
Linux Kernel Driver
    ↓
Hardware
```

처음에는 STM32에서 사용하던 HAL과 같은 이름이 등장해서,

> 여기서의 HAL을 STM32에서의 HAL과 같은 맥락으로 생각해도 되나?

라는 의문이 생겼다.

---

## 3. STM32 HAL과 Android HAL의 차이

두 HAL 모두 공통적으로 하드웨어 구현 차이를 상위 계층에서 숨긴다는 목적은 가지고 있다.

하지만 시스템에서 가지는 역할과 개발자가 접근하는 방식은 상당히 다르다.

### STM32 HAL

```text
Application
    ↓
HAL
    ↓
Register
    ↓
Hardware
```

주된 목적:

- Peripheral 사용 편의성 향상
- MCU family 간 API 통일
- Register-level programming 복잡성 감소

그리고 개발자는 필요하면 HAL 아래로 자유롭게 내려갈 수 있다.

```text
HAL_GPIO_WritePin()
        ↓
필요하면
        ↓
GPIOx->ODR 직접 접근
```

즉 STM32 HAL은 **접근 권한을 제한하는 경계라기보다 convenience abstraction에 가깝다.**

---

### Android HAL

Android에서는 구조가 더 크다.

```text
App
 ↓
Framework
 ↓
System Service
 ↓
HAL Interface
 ↓
Vendor HAL Implementation
 ↓
Kernel Driver
 ↓
Hardware
```

Android HAL은 단순 편의 라이브러리가 아니라,

> Android Framework와 Vendor Hardware Implementation 사이의 인터페이스 계약

에 가깝다.

예를 들어 스마트폰 제조사마다 실제 IMU가 다를 수 있다.

```text
Bosch IMU
TDK IMU
Samsung Sensor Hub
Qualcomm Sensor Subsystem
        ↑
    Vendor HAL
        ↑
 Android Sensors HAL Interface
        ↑
 Android Framework
```

Android Framework 입장에서는 아래쪽 하드웨어 구현이 무엇인지 몰라도 동일하게 accelerometer, gyroscope 등의 인터페이스를 사용할 수 있다.

즉 Android HAL은 다음 문제를 해결한다.

```text
Samsung hardware implementation
Pixel hardware implementation
Xiaomi hardware implementation
        ↓
        HAL
        ↓
공통 Android Framework API
```

---

## 4. Android HAL은 못 건드리게 만든 추상화?

처음에는 다음과 같이 이해할 수 있었다.

> STM32 HAL은 편하게 쓰려고 만든 추상화이고,
> Android HAL은 내부를 못 건드리게 막아두고 정해진 기능만 사용하도록 만든 추상화인가?

일반적인 Android App에서는 보통 다음과 같은 API를 통해서만 센서나 카메라에 접근한다.

```text
SensorManager
Camera2
Audio API
Bluetooth API
...
```

그리고 일반 앱이 다음 영역까지 직접 내려가는 것은 어렵다.

```text
IMU register 직접 변경
Sensor FIFO watermark 변경
IRQ priority 수정
Kernel sensor driver 변경
HAL thread scheduling 변경
Device Tree 변경
```

즉 실질적으로는 제조사가 제공한 인터페이스 범위 안에서만 하드웨어를 사용할 수 있다.

하지만 정확히 말하면 HAL 자체가 전부는 아니다.

Android에서는 여러 계층이 함께 접근을 제한한다.

```text
Application Sandbox
        +
Linux UID / Permission
        +
SELinux
        +
Binder Permission
        +
Framework API
        +
HAL Interface
        +
Kernel Permission
```

따라서 Android HAL을 단순히

> 아래를 못 건드리게 막는 추상화 계층

라고 정의하기 보다는

> Android의 OS 계층과 제조사별 하드웨어 구현을 분리하는 시스템 아키텍처 경계이며, 일반 앱은 Android 보안 및 권한 구조 때문에 그 아래에 직접 접근하기 어렵다.

라고 볼 수 있다.

---

## 5. Android IMU Sampling 문제와 연결

이번 HAL 개념을 조사하게 된 계기는 스마트폰 IMU sampling 문제였다.

Android에서 센서를 사용할 때 samplingPeriod를 설정할 수 있지만 이것은 RTOS의 strict periodic deadline과는 다르다.

예를 들어:

samplingPeriod = 5 ms

라고 요청해도 의미는 다음과 가깝다.

목표 sensor sampling interval ≈ 5 ms

이지,

이 application thread를 반드시 매 5 ms마다 실행

Deadline = 5 ms

를 보장하는 것은 아니다. 

센서 경로는 대략 다음과 같다.

```text
IMU Hardware
    ↓
Sensor FIFO / Sensor Hub
    ↓
Kernel Driver
    ↓
Vendor Sensors HAL
    ↓
SensorService
    ↓
Android Application
```

IMU 자체는 내부 ODR 기준으로 비교적 일정한 주기로 데이터를 측정할 수 있지만, 데이터가 Application까지 전달되는 시점에는 다음의 영향을 받는다.

- Linux scheduler
- Binder communication
- SensorService
- batching
- power management
- CPU load
- thermal throttling

따라서 Android Application에서 callback arrival timing 자체를 strict realtime으로 보장하기 어렵다.

---

## 6. Linux / RTOS와 비교했을 때의 의미

일반 Linux도 hard realtime OS가 아니기 때문에 strict deadline을 보장하지 않는다.

하지만 Linux에서는 직접 아래와 같은 방법을 적용할 수 있다.

```text
PREEMPT_RT
SCHED_FIFO
SCHED_RR
CPU affinity
IRQ affinity
CPU isolation
Kernel configuration
Driver modification
```

Jetson이나 직접 구성하는 Embedded Linux 시스템에서는 시스템 전체를 통제할 수 있기 때문에 이런 시도를 현실적으로 할 수 있다.

Android 역시 Linux kernel 기반이기 때문에 커널 수준에서 realtime tuning을 하는 것이 기술적으로 불가능한 것은 아니지만, 
실제 상용 스마트폰에서는 bootloader, vendor driver, HAL, SELinux, AVB, proprietary component 등의 문제까지 얽히므로 현실적인 제약이 따른다.