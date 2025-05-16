# 🎮 LED Remote Control – PWM Decoding with Arduino

> RC 리모컨과 수신기를 이용해 **RGB LED 및 일반 LED(2개)의 밝기, 색상, 전원 상태를 무선으로 제어**하는 프로젝트입니다.

---

## 🎥 시연 영상  
[https://youtu.be/l7083ztuV90?feature=shared)]

---

## 🧾 프로젝트 개요

이 프로젝트는 **아두이노**와 **PWM 신호 기반 RC 수신기(R9DS)**를 활용하여,  
RGB LED와 일반 LED의 색상 및 밝기를 **무선 리모컨으로 직관적으로 조절**할 수 있는 시스템입니다.

---

## ✨ 주요 기능

- 리모컨의 채널을 통해 다음과 같은 동작 가능:
  - `CH5`: 제어 대상 선택 (RGB / LED1 / LED2)
  - `CH6`: 밝기 조절
  - `CH7`: 색상 선택 (RGB에만 적용)
  - `CH8`: 전체 전원 ON/OFF
- **PinChangeInterrupt** 기반의 인터럽트로 PWM 신호를 실시간 감지
- `HSV → RGB 변환`, `감마 보정`을 통해 자연스러운 색상 및 밝기 출력

---

## 🔁 시스템 흐름도
```
[ 리모컨 CH5~CH8 입력 ]  
↓  
[ RC 수신기 (PWM 출력) ]  
↓  
[ Arduino (인터럽트 기반 PWM 수신) ]  
↓  
[ RGB + 일반 LED 제어 ]  
 ``` 


---

## ⚙️ 기술 스택

- **Arduino UNO**
- **RC 수신기: Radiolink R9DS**
- **RC 송신기: Radiolink AT9 / AT10 등**
- `C++ (Arduino Sketch)`
- [`PinChangeInterrupt`](https://github.com/NicoHood/PinChangeInterrupt) 라이브러리 사용

---

## 📦 설치 및 실행 방법

1. Arduino IDE 실행
2. 라이브러리 설치:  
   `Tools → Manage Libraries → PinChangeInterrupt` 검색 후 설치
3. `led_remote.ino` 스케치 열기
4. 회로 구성 후 업로드

### 📡 리모컨 채널 연결

| 채널 | 기능              | 아두이노 핀 |
|------|-------------------|-------------|
| CH5  | 제어 대상 선택     | D2          |
| CH6  | 밝기 조절          | D3          |
| CH7  | 색상(Hue) 전환     | D4          |
| CH8  | 전원 ON/OFF       | D7          |

---

## 🔌 회로 구성도
![car1](https://github.com/user-attachments/assets/40879dd0-a7c0-4480-93e9-86a0c50297c2)
![car2](https://github.com/user-attachments/assets/28de8ebd-7486-4f8a-9c33-70039ee06da6)



### 🎮 입력 (RC 수신기에서 PWM 입력)
핀 번호	기능 설명	아두이노 연결 핀
CH5	제어 대상 선택	-D2
CH6	밝기 조절	-D3
CH7	색상(Hue) 조절	-D4
CH8	전체 전원 ON/OFF	-D7

### 💡 출력 (LED 제어)
출력 대상	아두이노 핀	설명
RGB LED - Red	-D6	analogWrite PWM 출력
RGB LED - Green	-D10	analogWrite PWM 출력
RGB LED - Blue	-D11	analogWrite PWM 출력
일반 LED1	-D9	analogWrite PWM 출력
일반 LED2	-D5	analogWrite PWM 출력


### 🎨 회로도 다이어그램 (텍스트 기반)
  ```
          +-------------------------+  
          |        Arduino         |  
          |      (Uno / Nano)      |  
          +-------------------------+  
   PWM IN | D2  <--- CH5 (선택)     |  
          | D3  <--- CH6 (밝기)     |  
          | D4  <--- CH7 (색상)     |  
          | D7  <--- CH8 (전원)     |   
          | D6  ---> R (RGB LED) ------|>|--- GND  (3색 LED는 내부저항 존재)  
          | D10 ---> G (RGB LED) ------|>|--- GND  
          | D11 ---> B (RGB LED) ------|>|--- GND  
          | D9  ---> 일반 LED1 ----[330Ω]---|>|--- GND  
          | D5  ---> 일반 LED2 ----[330Ω]---|>|--- GND  
          | GND ----------------------------- GND (공통 접지)  
          +-------------------------+  
  ```

각 일반 LED에는 330Ω 저항을 직렬로 연결하여 과전류 방지.

입력 채널(CH5~CH8)은 RC 수신기에서 PWM 신호로 들어오며, 해당 핀들은 INPUT_PULLUP 모드로 설정되어 있어 외부에서 LOW 상태를 인식할 수 있음.

---

## 🔧 코드 구조 및 주석

각 기능은 다음과 같이 **작성자별로 역할 분담**하여 구현하였습니다.  
소스코드 내에 각 영역마다 주석을 통해 구현 내용을 명확히 표시하였습니다.

| 담당자   | 주요 구현 내용                                       |
|----------|------------------------------------------------------|
| 황주옥   | PWM 신호 입력 핀 설정, 인터럽트 처리, setup 구성     |
| 박기주   | 출력 대상 및 밝기 상태 관리, 색상 변화 및 출력 로직 |

--- 

## 📄 코드 설명 (LED PWM 제어 프로젝트)

> 이 문서는 본 프로젝트의 핵심 코드에 대한 매우 자세한 설명을 제공합니다. 각 함수가 수행하는 역할, 입력 및 출력, 동작 흐름을 기반으로 구조적으로 정리되어 있습니다.
```cpp
 🧩 1. 핀 설정

const int ch5_pin = 2;  // 제어 대상 선택 채널 (RGB, LED1, LED2)
const int ch6_pin = 3;  // 밝기 조절 채널
const int ch7_pin = 4;  // 색상(Hue) 조절 채널
const int ch8_pin = 7;  // 전원 ON/OFF 채널

const int r_pin = 6;    // RGB LED - Red 출력
const int g_pin = 10;   // RGB LED - Green 출력
const int b_pin = 11;   // RGB LED - Blue 출력
const int led1_pin = 9; // 일반 LED1 출력
const int led2_pin = 5; // 일반 LED2 출력

ch5~ch8_pin: RC 수신기에서 오는 PWM 신호를 입력받는 핀입니다. 각 채널은 특정 기능에 대응됩니다.

r/g/b/led1/led2_pin: analogWrite()를 통해 PWM 신호로 LED 밝기를 출력하는 핀입니다.

⏱️ 2. PWM 신호 측정 변수

volatile int pwm_chX = 1500;
volatile unsigned long start_chX = 0;

pwm_chX: 각 채널(CH5~CH8)의 펄스 폭(us)을 저장합니다.

start_chX: PWM 신호가 HIGH로 전환되었을 때의 시간 (micros())을 저장합니다.

volatile: 인터럽트 함수에서 수정되므로 최적화 방지를 위해 사용합니다.

⚙️ 3. setup() 함수

void setup() {
  Serial.begin(9600);
  pinMode(...);
  attachPCINT(...);
}

시리얼 출력 초기화

각 입력 핀에 내부 풀업 저항 활성화 (INPUT_PULLUP)

출력 핀은 OUTPUT으로 설정

attachPCINT()를 통해 각 입력 핀에 인터럽트 핸들러 등록 (핀 상태 변화 감지)

🔁 4. loop() 함수

void loop() {
  bool powerOn = pwm_ch8 > 1500;
  int mappedTarget = map(pwm_ch5, 1000, 2000, 0, 3);
  selectedTarget = constrain(mappedTarget, 0, 2);

  int brightness = getSmoothBrightness(pwm_ch6);

pwm_ch8 값이 1500us 이상이면 전원 ON 상태로 판단

pwm_ch5를 0(RGB), 1(LED1), 2(LED2)로 매핑하여 제어 대상 선택

getSmoothBrightness() 함수로 CH6 값을 부드러운 밝기로 변환

if (selectedTarget == 0) {
  float hue = map(pwm_ch7, 1200, 2000, 0, 360);
  hsvToRgb(hue, 1.0, 1.0, r_raw, g_raw, b_raw);
  brightness_r = r_raw * brightness / 255;
  ...
}

RGB가 선택된 경우, CH7 값(Hue)을 HSV로 변환하여 색상 출력값 계산

일반 LED1/LED2는 밝기만 조절함

if (powerOn) {
  analogWrite(...);
} else {
  analogWrite(..., 0);
}

전원 상태에 따라 LED 밝기를 설정하거나 모두 OFF 처리

⚡ 5. 인터럽트 핸들러

각 채널의 핀에서 상태 변화(CHANGE)를 감지하여 펄스 폭 계산

void ISR_chX() {
  if (digitalRead(chX_pin) == HIGH)
    start_chX = micros();
  else
    pwm_chX = micros() - start_chX;
}

HIGH 전환 시 시간 기록, LOW 전환 시 펄스 길이 계산하여 저장

🌈 6. 밝기 보정 함수 – getSmoothBrightness()

int getSmoothBrightness(int raw) {
  float norm = map(raw, 1000, 2000, 0, 1000) / 1000.0;
  float gamma = 1.2;
  int brightness = pow(norm, gamma) * 215 + 40;
  return constrain(brightness, 40, 255);
}

CH6의 PWM 입력을 0.0~1.0 사이로 정규화 후 감마 보정 적용

시각적으로 자연스럽고 선형적이지 않은 밝기 곡선을 생성

🎨 7. 색상 변환 함수 – hsvToRgb()

void hsvToRgb(float h, float s, float v, int& r, int& g, int& b) {
  float c = v * s;
  float x = c * (1 - fabs(fmod(h / 60.0, 2) - 1));
  float m = v - c;
  ...
  r = constrain(round((r1 + m) * 255), 0, 255);
}

Hue(0~360)를 기준으로 HSV 색상값을 RGB로 변환

색상 구간(060, 60120, ...)에 따라 r1/g1/b1 계산

최종적으로 RGB 값을 0~255로 스케일링 후 출력

🧪 8. 시리얼 디버깅 출력

Serial.print("[Selected] RGB | LED1 | LED2");
Serial.print("Brightness R,G,B: ...");

현재 선택된 제어 대상

RGB LED의 출력 밝기

LED1, LED2의 밝기

전원 상태 등을 실시간으로 출력

✅ 전체 흐름 요약

RC 리모컨에서 PWM 신호 입력 (CH5~CH8)

인터럽트로 각 채널의 펄스 폭 실시간 측정

loop() 함수에서 선택 대상, 색상, 밝기 계산

analogWrite()로 LED 출력

시리얼로 상태 출력


```

--- 

## ✅ 결론 및 기대 효과
이 프로젝트는 PWM 신호 기반 인터럽트를 통해 빠르고 정확한 무선 LED 제어 시스템을 구현합니다.

기존 pulseIn() 방식은 하나의 채널만 읽을 수 있는 한계가 있었으며,
여러 채널의 PWM 신호를 동시에 처리할 수 없었습니다.

이에 따라 인터럽트 기반 방식으로 전환함으로써

동시성 향상,

반응 속도 개선,

시스템 확장성 확보가 가능해졌습니다.


