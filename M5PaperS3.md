# M5PaperS3 — standalone code

**Board enum:** `board_M5PaperS3` (19)
**SoC:** ESP32-S3

Successor to M5Paper. Uses a **parallel 8-bit EPD interface** (no IT8951 controller — data lanes go directly to the panel).

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| Parallel 8-bit e-paper (960×540) | Yes | M5GFX (ONLY — parallel EPD is not in Adafruit_EPD) |
| GT911 cap-touch | Yes | GT911 driver |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| Buzzer | Yes, GPIO21 | ledcWrite |
| MicroSD | Yes | `SD.h` |
| Battery + charger | Yes | ADC |
| Power hold | GPIO44 | digitalWrite |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold** | **44** |
| EPD data D0-D7 | 6, 14, 7, 12, 9, 11, 8, 10 |
| EPD control (CKV, SPV, OE, LE, etc.) | dedicated — see M5GFX source |
| Internal I²C SCL / SDA | 42 / 41 |
| Grove (Port A) SCL / SDA | 1 / 2 |
| SD SPI SCLK/MOSI/MISO/CS | 39 / 38 / 40 / 47 |
| Buzzer | 21 |

---

## 1. Power hold + display

```cpp
#include <M5GFX.h>
M5GFX epd;
void setup() {
  pinMode(44, OUTPUT); digitalWrite(44, HIGH);   // LATCH
  epd.init();
  epd.setEpdMode(epd_mode_t::epd_fastest);
}
```

Parallel EPD is timing-critical — unless you want to port Espressif's `esp_lcd_panel_epd` stack yourself, **use M5GFX**. The panel init lives in [M5GFX.cpp:1411](../../M5GFX/src/M5GFX.cpp#L1411) and [Panel_EPD.hpp](../../M5GFX/src/lgfx/v1/platforms/esp32/Panel_EPD.hpp).

---

## 2. Touch / RTC / Buzzer / SD

Same patterns as [M5Paper.md](M5Paper.md): GT911 I²C, PCF8563 RTC, ledcWrite buzzer, `SD.h`.

---

## Brightness / Volume

E-paper — no backlight. Buzzer on GPIO21, `magnification = 48`:
```cpp
M5.Speaker.setVolume(128);
M5.Speaker.tone(2000, 200);
```

---

## Power management

ADC on **GPIO3** with 2:1 divider, **plus a dedicated charge-status pin on GPIO4** (LOW = charging).

### With M5Unified

```cpp
int  pct = M5.Power.getBatteryLevel();       // 0..100 from voltage
int  mv  = M5.Power.getBatteryVoltage();
auto chg = M5.Power.isCharging();             // reliable on PaperS3 — reads GPIO4
```

### Standalone

```cpp
constexpr int VBAT_PIN = 3, CHG_STAT_PIN = 4;
void setup() {
  pinMode(CHG_STAT_PIN, INPUT);                // charge status
  pinMode(44, OUTPUT); digitalWrite(44, HIGH); // LATCH
}

float vbat_mv() { return (analogRead(VBAT_PIN) * 3300.0f / 4095.0f) * 2.0f; }
bool  charging() { return digitalRead(CHG_STAT_PIN) == LOW; }
int   pct_from_mv(float mv) {
  int p = (mv - 3300) * 100 / (4150 - 3350);
  return (p < 0) ? 0 : (p > 100) ? 100 : p;
}
```

Shutdown: `digitalWrite(44, LOW)`.

---

## Source references

| Topic | File : line |
|-------|-------------|
| PaperS3 autodetect | [M5GFX.cpp:1411](../../M5GFX/src/M5GFX.cpp#L1411) |
| Parallel EPD panel driver | [M5GFX/src/lgfx/v1/platforms/esp32/Panel_EPD.hpp](../../M5GFX/src/lgfx/v1/platforms/esp32/Panel_EPD.hpp) |
| PaperS3 pinmap | [src/M5Unified.cpp:2080](src/M5Unified.cpp#L2080) |
