# M5Station — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5Station` (9)
**SoC:** ESP32-D0WDQ6

DIN-rail industrial enclosure with a small ST7789 LCD and **AXP192 PMIC**. Exposes 2× Port B (B + B2) and 2× Port C (C + C2) for field wiring.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD 135×240 | Yes, PMIC-backlit | M5GFX |
| AXP192 PMIC | Yes, I²C `0x34` | AXP192 driver |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| WS2812 LED | Yes, GPIO4 | Adafruit_NeoPixel |
| Speaker | HAT only | — |
| BtnA / BtnB / BtnC | Yes, GPIO37 / 38 / 39 | digitalRead |
| Battery | Yes (via AXP192) | AXP192 battery reg |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / SCLK | 23 / 18 |
| LCD CS / DC / RST | 5 / 19 / 15 |
| **LCD backlight** | **AXP192 LDO3** (must enable) |
| Internal I²C SCL / SDA | 22 / 21 (PMIC + RTC) |
| Grove (Port A) SCL / SDA | 33 / 32 |
| Port B pin1/pin2 | 35 / 25 |
| Port B2 (a.k.a. Port D) | 36 / 26 |
| Port C pin1/pin2 | 13 / 14 |
| Port C2 (a.k.a. Port E) | 16 / 17 |
| WS2812 LED | 4 |
| BtnA / BtnB / BtnC | 37 / 38 / 39 |

---

## ⚠️ LCD backlight needs AXP192 LDO3

Unlike Core2 (DCDC3), Station uses **LDO3** for backlight:

```cpp
Wire.begin(21, 22, 400000);
// LDO3 = 3.3 V:
uint8_t ldo23 = read(0x28);
ldo23 = (ldo23 & 0xF0) | 0x0F;    // LDO3 nibble = 3.3 V
write(0x28, ldo23);
write(0x12, read(0x12) | (1<<3)); // enable LDO3
```

---

## 1. Display

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() { lcd.init(); lcd.setBrightness(128); }
```

Autodetect: [M5GFX.cpp:906](../../M5GFX/src/M5GFX.cpp#L906).

---

## 2. Port B/C/D/E wiring

Each port exposes 2 GPIO + 5V + GND on a Grove-style connector. Use any as `digitalRead`/`digitalWrite`/`analogRead` (pins are regular GPIOs). No PMIC gating on ports — 5V rail is always live (or toggle via `EXTEN` if you care about power draw on idle ports).

---

## 3. RTC / AXP192 / LED / Buttons

- RTC: PCF8563 @ `0x51`.
- AXP192 `0x34` — see [M5StackCore2.md](M5StackCore2.md) for init template.
- LED: `Adafruit_NeoPixel(1, 4, NEO_GRB + NEO_KHZ800)`.
- Buttons: GPIO37/38/39 `INPUT_PULLUP`.

---

## Brightness (LCD backlight)

**AXP192 LDO3** — identical mechanism to [M5Tough](M5Tough.md#brightness-lcd-backlight). Copy the `tough_brightness()` helper.

## Volume

No onboard speaker. With SPK HAT: see [M5StackCoreInk § Volume](M5StackCoreInk.md#volume-buzzer) for the HAT-speaker enable + duty volume pattern.

---

## Power management

**AXP192** at I²C `0x34`. Same API + registers as StickC — see [M5StickC.md § Standalone AXP192 reads](M5StickC.md#standalone-axp192-reads).

The AXP192 on Station also drives **Port A 5V output** via EXTEN:
```cpp
M5.Power.setExtOutput(true);          // turn on 5V to Grove ports
axp192_mod(0x12, 0x40, 0x40);         // standalone: reg 0x12 bit 6
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| Station autodetect | [M5GFX.cpp:906](../../M5GFX/src/M5GFX.cpp#L906) |
| Station pinmap | [src/M5Unified.cpp:1607](src/M5Unified.cpp#L1607) |
