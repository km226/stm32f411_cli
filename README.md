# STM32F411 CLI

STM32F411RE (ARM Cortex-M4) 환경에서 동작하는 **FreeRTOS 기반 임베디드 CLI 셸**.
UART 시리얼 터미널로 명령어를 입력받아 GPIO 제어, 메모리 덤프, 온도 측정 등을
수행하고, 센서 값을 주기적으로 스트리밍하는 모니터링 프로토콜을 제공한다.

---

## 주요 특징

- **FreeRTOS 멀티태스킹** — CLI, LED, 온도 측정, 모니터링을 4개 태스크로 분리하고
  뮤텍스와 메시지 큐로 자원을 보호
- **인터랙티브 CLI** — 명령어 히스토리(↑/↓), 백스페이스, `Ctrl+C` 인터럽트,
  ANSI 이스케이프 시퀀스 파싱을 직접 구현
- **런타임 명령어 등록** — `cliAdd()`로 함수 포인터를 테이블에 등록하는 구조.
  새 명령어 추가 시 파서 수정이 불필요
- **계층형 아키텍처** — `ap` / `hw` / `bsp` / `common` 4계층으로 분리해
  애플리케이션 로직과 하드웨어 의존성을 격리
- **레벨별 로깅 시스템** — 6단계 로그 레벨, ANSI 컬러 출력, 모듈별 태그,
  컴파일 타임 + 런타임 이중 필터링
- **텔레메트리 프로토콜** — 센서 값을 ID/타입/값 노드로 패킹해 주기 전송
- **CMake + Ninja 빌드** — CubeIDE 없이 CLI만으로 크로스 컴파일 가능

---

## 개발 환경

| 항목 | 내용 |
|---|---|
| MCU | STM32F411RE (Cortex-M4, 84 MHz) |
| 보드 | NUCLEO-F411RE |
| RTOS | FreeRTOS (CMSIS-RTOS v2 API) |
| 툴체인 | GNU Arm Embedded (`arm-none-eabi-gcc`) |
| 빌드 | CMake + Ninja (CMakePresets) |
| 에디터 | VS Code + clangd / STM32CubeIDE |
| 코드 생성 | STM32CubeMX (`stm32f411_cli.ioc`) |

---

## 빌드 및 실행

### 사전 준비

```bash
# Ubuntu / WSL
sudo apt install cmake ninja-build gcc-arm-none-eabi openocd
```

### 빌드

```bash
cmake --preset Debug        # 또는 Release
cmake --build --preset Debug
```

빌드 산출물은 `build/Debug/`에 생성된다. 툴체인 설정은
`cmake/gcc-arm-none-eabi.cmake`에 정의되어 있으며, clang 툴체인
(`cmake/starm-clang.cmake`)도 준비되어 있다.

### 플래싱

```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
        -c "program build/Debug/stm32f411_cli.elf verify reset exit"
```

### 접속

ST-Link 가상 COM 포트로 접속한다. **보드레이트 9600**.

```bash
screen /dev/ttyACM0 9600      # Linux / macOS
# 또는 PuTTY, Tera Term (Windows)
```

접속하면 `CLI> ` 프롬프트가 표시된다.

---

## 명령어

터미널에서 `help`를 입력하면 등록된 명령어 목록을 볼 수 있다.

### `led` — LED 제어

```
led on                  # LED 켜기
led off                 # LED 끄기
led toggle              # 1회 토글
led toggle [period]     # period(ms) 주기로 자동 토글 (별도 태스크에서 수행)
```

### `gpio` — 범용 GPIO 제어

포트/핀을 문자열로 받아 런타임에 클럭을 활성화하고 설정한다.

```
gpio read a5            # PA5 입력 읽기
gpio write b12 1        # PB12를 HIGH로 출력
```

포트 `a`~`h`, 핀 `0`~`15` 지원.

### `md` — 메모리 덤프

지정 주소부터 16진수 + ASCII로 덤프한다. 접근 전 주소 유효성을 검사해
Flash / SRAM / System Memory / Peripheral 영역만 허용하므로,
잘못된 주소로 인한 HardFault를 방지한다.

```
md 08000000 32          # Flash 시작 주소부터 32바이트
md 20000000             # SRAM (길이 생략 시 16바이트)
```

```
0x08000000 : 00 20 00 20 C1 01 00 08 ... | . . ..........
```

### `temp` — MCU 내부 온도 센서

ADC1 + DMA(순환 모드)로 내장 온도 센서를 읽는다.

```
temp                    # 1회 측정
temp [period]           # period(ms) 주기로 자동 측정
```

### `info` — 시스템 정보

```
info                    # 모델, 펌웨어 버전, 빌드 일시, 96비트 UID, Device ID
info uptime             # 부팅 후 경과 시간 (ms)
```

### `log` — 로그 레벨 제어

```
log get                 # 현재 레벨 조회
log set [0~5]           # 레벨 변경
```

`0:FATAL  1:ERROR  2:WARN  3:INFO  4:DEBUG  5:VERBOSE`

### `mon` — 모니터링 모드

센서 값을 주기적으로 패킷 형태로 스트리밍한다. 활성화하면 LED·온도 태스크의
주기가 모니터 주기에 자동 동기화되고, 개별 태스크의 텍스트 출력은 억제된다.

```
mon on                  # 모니터링 시작 (기본 1000ms)
mon on [period]         # 주기 지정 (최소 10ms)
mon off                 # 중지 및 텍스트 모드 복귀
```

### `button` — 버튼 인터럽트 리포트

```
button on / off         # B1(PC13) EXTI 이벤트 로그 출력 토글
button                  # 현재 상태 조회
```

### `sys`, `cls`, `help`

```
sys reset               # NVIC_SystemReset()
cls                     # 화면 지우기
help                    # 명령어 목록
```

---

## 모니터링 프로토콜

`mon on` 상태에서 UART로 아래 ASCII 포맷을 주기 전송한다. 호스트 측
시각화 도구나 스크립트에서 파싱하기 쉽도록 설계했다.

```
$<노드수>,<ID>:<타입>:<값>,<ID>:<타입>:<값>...#
```

**예시**

```
$2,50:3:1,10:2:24.53#
```

→ 노드 2개. `ID 50`(LED 상태, `TYPE_BOOL`) = 1, `ID 10`(외부 온도, `TYPE_FLOAT`) = 24.53

**데이터 타입**

| 값 | 타입 |
|---|---|
| 0 | `TYPE_UINT8` |
| 1 | `TYPE_INT32` |
| 2 | `TYPE_FLOAT` |
| 3 | `TYPE_BOOL` |

**센서 ID 대역**

| 범위 | 용도 | 예시 |
|---|---|---|
| 0~9 | 시스템 공통 | `ID_SYS_HEARTBEAT`, `ID_SYS_UPTIME` |
| 10~29 | 환경 센서 | `ID_ENV_TEMP`, `ID_ENV_HUMI` |
| 30~49 | 사용자 입력 | `ID_IN_BUTTON_1` |
| 50~69 | 액추에이터 상태 | `ID_OUT_LED_STATE` |
| 70~89 | 모션/위치 | `ID_IMU_ACCEL_X` |
| 100+ | 에러/알람 | `ID_ALARM_CRITICAL` |

바이너리 모드(STX `0x02` / ETX `0x03`, XOR 체크섬)용 구조체와 상수도
정의되어 있어 확장 가능하다.

---

## FreeRTOS 태스크 구성

| 태스크 | 우선순위 | 스택 | 역할 |
|---|---|---|---|
| `defaultTask` | Normal | 2 KB | `apInit()` 후 CLI 메인 루프 |
| `myTaskLed` | Low | 1 KB | LED 자동 토글 및 상태 보고 |
| `myTaskTemp` | Low | 2 KB | 온도 주기 측정 및 보고 |
| `myTaskMonitor` | Low | 2 KB | 모니터링 패킷 송신 |

**동기화**

- UART 수신은 인터럽트 콜백에서 **메시지 큐**(256 depth)에 적재하고,
  CLI 태스크가 `osMessageQueueGet()`으로 블로킹 대기한다. 폴링이 없어
  유휴 시 CPU를 점유하지 않는다.
- UART 송신은 **뮤텍스**로 보호해 여러 태스크의 출력이 섞이지 않게 한다.
- 모니터 패킷 버퍼도 뮤텍스로 보호한다.

**태스크 간 결합 분리**

`monitor`는 `ap` 계층을 직접 호출하지 않고 콜백(`monitorSetSyncHandler`)을 통해
주기 동기화를 요청한다. CLI의 `Ctrl+C` 처리도 같은 방식
(`cliSetCtrlHandler`)으로 상위 계층에 위임한다.

---

## 프로젝트 구조

```
stm32f411-cli/
├── MyApp/                      # 직접 작성한 애플리케이션 코드
│   ├── ap/                     #   애플리케이션 계층
│   │   ├── ap.c                #     CLI 명령어 구현, FreeRTOS 태스크 본체
│   │   └── monitor.c           #     텔레메트리 프로토콜
│   ├── hw/                     #   하드웨어 추상화 계층
│   │   ├── hw.c                #     주변장치 일괄 초기화
│   │   └── driver/
│   │       ├── cli.c           #       명령어 파서, 히스토리, ANSI 처리
│   │       ├── uart.c          #       큐/뮤텍스 기반 UART 래퍼
│   │       ├── log.c           #       로그 레벨 제어
│   │       ├── my_gpio.c       #       런타임 GPIO 제어
│   │       ├── temp.c          #       ADC+DMA 온도 센서
│   │       ├── led.c
│   │       └── button.c        #       EXTI 콜백
│   ├── bsp/                    #   보드 지원 (millis 등)
│   └── common/                 #   공통 정의
│       ├── def.h
│       ├── hw_def.h
│       └── log_def.h           #     로깅 매크로
│
├── Core/                       # CubeMX 생성 코드
│   ├── Inc/
│   └── Src/                    #   main.c, freertos.c, adc.c, usart.c ...
├── Drivers/                    # STM32 HAL / CMSIS
├── Middlewares/Third_Party/    # FreeRTOS
├── cmake/                      # 툴체인 파일
│   ├── gcc-arm-none-eabi.cmake
│   └── starm-clang.cmake
├── CMakeLists.txt
├── CMakePresets.json
├── STM32F411XX_FLASH.ld        # 링커 스크립트
└── stm32f411_cli.ioc           # CubeMX 프로젝트
```

---

## 하드웨어 구성

| 주변장치 | 설정 | 용도 |
|---|---|---|
| USART2 | 9600 bps, 인터럽트 수신 | CLI 콘솔 (ST-Link VCP) |
| ADC1 | `ADC_CHANNEL_TEMPSENSOR`, 84 cycles | 내부 온도 센서 |
| DMA2 Stream0 | 순환 모드, Half-word | ADC 값 자동 갱신 |
| GPIO PA5 | 출력 | LD2 사용자 LED |
| GPIO PC13 | EXTI Falling | B1 사용자 버튼 |

---

## 명령어 추가 방법

`MyApp/ap/ap.c`에 핸들러를 작성하고 `apInit()`에서 등록하면 된다.
파서나 디스패처는 수정할 필요가 없다.

```c
void cliMyCmd(uint8_t argc, char **argv)
{
    if (argc < 2) {
        cliPrintf("Usage: mycmd [arg]\r\n");
        return;
    }
    cliPrintf("arg = %s\r\n", argv[1]);
}

void apInit(void)
{
    // ...
    cliAdd("mycmd", cliMyCmd);
}
```

최대 32개 명령어, 인자 4개, 입력 라인 32바이트까지 지원한다
(`MyApp/hw/driver/cli.c` 상단 상수로 조정 가능).
