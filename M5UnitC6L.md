# M5UnitC6L — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5UnitC6L` (25)
**SoC:** ESP32-C6

Unit-sized enclosure with a **SSD1306 64×48 OLED** and a **PI4IOE5V6408 I/O expander** for button + extra IO.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| SSD1306 64×48 OLED | Yes, SPI | M5GFX OR Adafruit_SSD1306 |
| PI4IOE5V6408 I/O expander | Yes, I²C `0x43` internal bus | raw Wire + register writes |
| Piezo buzzer | Yes, GPIO11 | ledcWrite |
| WS2812 LED | Yes, GPIO2 | Adafruit_NeoPixel |
| BtnA | Yes (via I/O expander reg `0x0F` bit 0) | I²C read |

No display-touch, no speaker codec, no IMU, no RTC, no battery.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| OLED MOSI / SCLK | 21 / 20 |
| OLED CS / RST | 6 / 15 |
| OLED DC | not a GPIO — I²C-only control? | (check schematic — M5GFX drives DC via a side channel) |
| Internal I²C SCL / SDA | 8 / 10 (I/O expander) |
| Port B / C pin1 / pin2 | 4 / 5 |
| Buzzer | 11 |
| WS2812 | 2 |

---

## 1. Display (SSD1306 via M5GFX)

```cpp
#include <M5GFX.h>
M5GFX oled;
void setup() {
  oled.init();
  oled.setTextColor(0xFFFF);
  oled.drawString("hello", 0, 0);
}
```

Autodetect: [M5GFX.cpp:2201](../../M5GFX/src/M5GFX.cpp#L2201) (shared with ArduinoNessoN1).

---

## 2. I/O expander (PI4IOE5V6408 @ `0x43`)

The expander holds the button state and controls backlight / status lines. Standalone read:

```cpp
#include <Wire.h>
void setup() {
  Wire.begin(10 /* SDA */, 8 /* SCL */, 400000);
}
uint8_t pi4io_read(uint8_t reg) {
  Wire.beginTransmission(0x43);
  Wire.write(reg);
  Wire.endTransmission(false);
  Wire.requestFrom(0x43, 1);
  return Wire.read();
}
void pi4io_write(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(0x43);
  Wire.write(reg); Wire.write(val);
  Wire.endTransmission();
}

// BtnA state: reg 0x0F, bit 0 (inverted — 0 = pressed)
bool btnA_pressed() { return !(pi4io_read(0x0F) & 0x01); }
```

Full class: [src/utility/PI4IOE5V6408_Class.cpp](src/utility/PI4IOE5V6408_Class.cpp).

Register map summary:
- `0x03` = direction (1=output)
- `0x05` = output value
- `0x07` = push-pull / open-drain
- `0x0B` = pull-up/down config
- `0x0F` = input value

---

## 3. Buzzer / LED

GPIO11 buzzer (ledcWrite), GPIO2 WS2812 (Adafruit_NeoPixel).

---

## Brightness (SSD1306 OLED contrast)

Not really "backlight" — an OLED is self-emissive. SSD1306's equivalent is the **contrast register `0x81`** sent as command over the SPI bus (reg 0x81 + 1 byte 0..0xFF). M5GFX maps `setBrightness(0..255)` to this register directly.

```cpp
// standalone (SPI command mode for SSD1306)
// send: 0x81, <brightness>
// — done via the panel driver, not a separate bus
```

With M5GFX: `lcd.setBrightness(0..255)` — 0 = fully off, 255 = max brightness.

## Volume (buzzer)

GPIO11 buzzer, `magnification = 48`. Standard path.

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| UnitC6L autodetect | [M5GFX.cpp:2201](../../M5GFX/src/M5GFX.cpp#L2201) |
| UnitC6L pinmap | [src/M5Unified.cpp:1467](src/M5Unified.cpp#L1467) |
| PI4IOE5V6408 driver | [src/utility/PI4IOE5V6408_Class.cpp](src/utility/PI4IOE5V6408_Class.cpp) |
