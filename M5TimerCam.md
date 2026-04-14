# M5TimerCam — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5TimerCam` (132)
**SoC:** ESP32-D0WDQ6 + 8 MB PSRAM
**Camera:** OV3660 (3 MP) — **not** managed by M5Unified, you drive it via `esp32-camera`

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| OV3660 camera | Yes | [esp32-camera](https://github.com/espressif/esp32-camera) — `esp_camera_init()` |
| BM8563 (PCF8563) RTC | Yes, I²C `0x51` internal bus | Adafruit_PCF8563 or raw Wire |
| Charger IC (battery) | Yes | read battery via ADC / standalone charger |
| Green LED | Yes, GPIO2 | Arduino digitalWrite |
| Power hold | **GPIO33 must be HIGH** | Arduino digitalWrite |

No display, no speaker/mic, no IMU.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold (latch 3.3V on battery)** | **33** (HIGH = stay on) |
| Green LED | 2 |
| Internal I²C SCL / SDA | 14 / 12 |
| Grove (Port A) SCL / SDA | 13 / 4 |
| Camera pins | handled by `esp32-camera` — see below |
| Battery voltage ADC | 38 (via divider) |

### Camera pins (for `camera_config_t`)

These are the pins the OV3660 connects to — wire them into `esp_camera_init()`:

```
CAM_PIN_PWDN     = -1    // not connected
CAM_PIN_RESET    = 15
CAM_PIN_XCLK     = 27
CAM_PIN_SIOD     = 25    // SCCB SDA (camera's I²C)
CAM_PIN_SIOC     = 23    // SCCB SCL
CAM_PIN_D7       = 19
CAM_PIN_D6       = 36
CAM_PIN_D5       = 18
CAM_PIN_D4       = 39
CAM_PIN_D3       = 5
CAM_PIN_D2       = 34
CAM_PIN_D1       = 35
CAM_PIN_D0       = 32
CAM_PIN_VSYNC    = 22
CAM_PIN_HREF     = 26
CAM_PIN_PCLK     = 21
```

(Cross-check against the TimerCam schematic — these are the commonly-published mappings; the M5Stack `m5-timer-cam` repo is authoritative.)

---

## 1. Power hold (critical on battery)

```cpp
void setup() {
  pinMode(33, OUTPUT);
  digitalWrite(33, HIGH);    // keeps 3.3V alive without USB
  // ... rest of your code
}
```

To power off: `digitalWrite(33, LOW);` and the 3.3V regulator cuts.

---

## 2. Camera init (standalone)

```cpp
#include "esp_camera.h"
static camera_config_t cfg = {
  .pin_pwdn = -1, .pin_reset = 15,
  .pin_xclk = 27, .pin_sscb_sda = 25, .pin_sscb_scl = 23,
  .pin_d7 = 19, .pin_d6 = 36, .pin_d5 = 18, .pin_d4 = 39,
  .pin_d3 = 5, .pin_d2 = 34, .pin_d1 = 35, .pin_d0 = 32,
  .pin_vsync = 22, .pin_href = 26, .pin_pclk = 21,
  .xclk_freq_hz = 20000000,
  .ledc_timer = LEDC_TIMER_0, .ledc_channel = LEDC_CHANNEL_0,
  .pixel_format = PIXFORMAT_JPEG,
  .frame_size = FRAMESIZE_QVGA,
  .jpeg_quality = 12,
  .fb_count = 2,
  .fb_location = CAMERA_FB_IN_PSRAM,
  .grab_mode = CAMERA_GRAB_WHEN_EMPTY,
};

void setup() {
  pinMode(33, OUTPUT); digitalWrite(33, HIGH);
  esp_camera_init(&cfg);
  camera_fb_t *fb = esp_camera_fb_get();   // grab frame
  // ... process fb->buf (JPEG) ...
  esp_camera_fb_return(fb);
}
```

---

## 3. BM8563 RTC (PCF8563-compatible)

```cpp
#include <Wire.h>
#define RTC_ADDR 0x51
void setup() {
  Wire1.begin(12 /* SDA */, 14 /* SCL */, 400000);
  Wire1.beginTransmission(RTC_ADDR);
  Wire1.write(0x00);           // control reg 1
  Wire1.write(0x00);           // normal mode
  Wire1.endTransmission();
}
```

Copy the full PCF8563 driver from [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp).

---

## 4. Battery voltage

GPIO38 reads the battery voltage through a 2:1 divider. Convert:
```cpp
float vbat = (analogRead(38) / 4095.0) * 3.3 * 2.0;  // volts
```

---

## Power management

Simple charger IC. ADC on **GPIO38**, with unusual ratio **1.513** (not a standard 2:1 divider):

```cpp
float vbat_mv() {
  int raw = analogRead(38);
  return (raw * 3300.0f / 4095.0f) * 1.513f;      // mV
}
int pct_from_mv(float mv) {
  int p = (mv - 3300) * 100 / (4150 - 3350);
  return (p < 0) ? 0 : (p > 100) ? 100 : p;
}
```

With M5Unified:
```cpp
int pct = M5.Power.getBatteryLevel();
int mv  = M5.Power.getBatteryVoltage();
// isCharging() → charge_unknown (no status pin)
```

### Shutdown (crucial for battery use)

GPIO33 is the power-hold latch. Pull LOW to shut off:
```cpp
digitalWrite(33, LOW);
```

Common pattern: wake on RTC, snap photo, upload, pull GPIO33 LOW.

---

## Source references

| Topic | File : line |
|-------|-------------|
| TimerCam pin assignment (RTC only — camera is elsewhere) | [src/M5Unified.cpp:1605](src/M5Unified.cpp) |
| PCF8563 driver | [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp) |
| Camera example | [M5Stack/M5-TimerCAM](https://github.com/m5stack/M5-TimerCAM) (external repo) |
