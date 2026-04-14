# M5DinMeter — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5DinMeter` (13)
**SoC:** ESP32-S3

DIN-rail enclosed unit — rectangular 1.14" ST7789 LCD + rotary encoder. Same SoC footprint as [M5Dial](M5Dial.md); only the display panel differs.

---

## Pin map (full)

Identical to M5Dial **except panel**:

| Purpose | GPIO |
|---------|------|
| **Power hold** | **46** (same latch as Dial — assert HIGH first) |
| LCD MOSI / SCLK | 5 / 6 |
| LCD CS / DC / RST | 7 / 4 / 8 |
| LCD BL | 9 (LEDC ch7 PWM @256 Hz — note: lower freq than Dial) |
| Internal I²C SCL / SDA | 12 / 11 (RTC) |
| Grove (Port A) SCL / SDA | 15 / 13 |
| Port B pin1 / pin2 | 1 / 2 |
| Buzzer | 3 |
| WS2812 | 21 |
| Encoder A / B / press | 40 / 41 / 42 |
| Side button | 0 |

---

## 1. Display (ST7789 135×240)

### M5GFX autodetect

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  pinMode(46, OUTPUT); digitalWrite(46, HIGH);
  lcd.init();
}
```

Autodetect: [M5GFX.cpp:1600](../../M5GFX/src/M5GFX.cpp#L1600).

### Explicit LovyanGFX (compact)

Same bus config as [M5Dial](M5Dial.md) but replace `Panel_GC9A01` with `Panel_ST7789`, set:
- `panel_width = 135`, `panel_height = 240`
- `offset_x = 52`, `offset_y = 40` (ST7789 240×240 panel, displayed window 135×240 offset)
- `invert = true`
- backlight freq 256 Hz instead of 44100

DinMeter has **no touch** (unlike Dial). Drop the `Touch_FT5x06` block from the Dial config.

---

## 2. Rotary encoder

Same as Dial: `ESP32Encoder` on A=40, B=41, press=42.

---

## 3. Buzzer / RTC / Buttons

Identical to Dial — buzzer on GPIO3, PCF8563 at I²C `0x51` on internal bus (SDA=11, SCL=12), buttons on GPIO42 + GPIO0.

See [M5Dial.md](M5Dial.md) for the copy-paste snippets.

---

## Minimal sketch

Identical to Dial's minimal sketch — only difference is the screen is rectangular.

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO9, LEDC ch7, 256 Hz, 9-bit**.

```cpp
void bl_init() { ledcSetup(7, 256, 9); ledcAttachPin(9, 7); }
void bl_set(uint8_t b) { ledcWrite(7, b << 1); }
```

## Volume (buzzer)

GPIO3 buzzer, `magnification = 48`. Identical to Dial.

---

## Power management

ADC on **GPIO10** with 2:1 divider:
```cpp
float vbat_mv() {
  int raw = analogRead(10);
  return (raw * 3300.0f / 4095.0f) * 2.0f;
}
```

Same pct formula as CoreInk/Paper. `isCharging()` returns `charge_unknown` (no status pin). Shutdown: GPIO46 LOW.

---

## Source references

| Topic | File : line |
|-------|-------------|
| DinMeter autodetect + panel config | [M5GFX.cpp:1600](../../M5GFX/src/M5GFX.cpp#L1600) |
| DinMeter pinmap | [src/M5Unified.cpp:1681](src/M5Unified.cpp#L1681) |
