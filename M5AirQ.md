# M5AirQ — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AirQ` (15)
**SoC:** ESP32-S3

Air-quality monitor with e-paper display. Air-sensor chip (SEN55 / SGP41 / SHT40 etc.) lives on the **Grove/external I²C bus**; M5Unified does not touch it.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| 200×200 e-paper (GDEW0154D67 / M09 auto-detected) | Yes | M5GFX OR Adafruit_EPD with custom panel |
| Piezo buzzer | Yes, GPIO9 | ledcWrite |
| PCF8563 RTC | Yes, I²C `0x51` internal bus | Wire |
| WS2812 LED | Yes, GPIO21 | Adafruit_NeoPixel |
| Air sensor (on ext I²C) | Yes | sensor-specific — `SensirionI2CSen5x` or similar |
| Battery | Yes | ADC |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold** | **46** |
| EPD MOSI / SCLK | 6 / 5 |
| EPD CS / DC / RST | 4 / 3 / 2 |
| EPD BUSY | 1 |
| Internal I²C SCL / SDA | 12 / 11 |
| External I²C (sensor) SCL / SDA | 15 / 13 |
| Buzzer | 9 |
| WS2812 | 21 |
| BtnA | 0 |
| BtnB | 8 |

---

## 1. E-paper display

Don't refresh faster than once per ~1 s on full refresh (the panel wears with writes).

```cpp
#include <M5GFX.h>
M5GFX epd;
void setup() {
  pinMode(46, OUTPUT); digitalWrite(46, HIGH);
  epd.init();
  epd.setEpdMode(epd_mode_t::epd_fastest);  // OR epd_quality for full gray
  epd.fillScreen(TFT_WHITE);
  epd.setTextColor(TFT_BLACK);
  epd.drawString("Air Quality", 20, 20);
}
```

Autodetect: [M5GFX.cpp:1755](../../M5GFX/src/M5GFX.cpp#L1755).

---

## 2. Air sensor

The SEN55 (or whichever variant your unit has) is on the **external** bus. Example with SEN55:

```cpp
#include <SensirionI2CSen5x.h>
SensirionI2CSen5x sen5x;
void setup() {
  Wire.begin(13 /* SDA */, 15 /* SCL */, 100000);
  sen5x.begin(Wire);
  sen5x.startMeasurement();
}
```

---

## 3. Buzzer / RTC / LED / Buttons

Same patterns as [M5Dial.md](M5Dial.md) — buzzer PWM, PCF8563 on internal bus, WS2812, GPIO-read buttons.

---

## Brightness / Volume

E-paper — no backlight. Buzzer on GPIO9, `magnification = 48`:
```cpp
M5.Speaker.setVolume(128);
M5.Speaker.tone(2000, 200);
```

---

## Power management

ADC on **GPIO14 (ADC2)** with 2:1 divider. Same pct formula as CoreInk/Paper. No charge-status pin. Shutdown: GPIO46 LOW.

```cpp
int pct = M5.Power.getBatteryLevel();
int mv  = M5.Power.getBatteryVoltage();
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| AirQ autodetect | [M5GFX.cpp:1755](../../M5GFX/src/M5GFX.cpp#L1755) |
| AirQ pinmap | [src/M5Unified.cpp:1663](src/M5Unified.cpp#L1663) |
