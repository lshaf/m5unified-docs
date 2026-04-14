# ArduinoNessoN1 — standalone code

**Board enum:** `board_ArduinoNessoN1` (23)
**SoC:** ESP32-C6

Custom Arduino-partnership board: **ST7789 135×240 LCD** + piezo buzzer + **2× PI4IOE5V6408** I/O expanders (buttons + backlight).

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD | Yes, SPI | M5GFX OR LovyanGFX |
| **Backlight via I/O expander #2** (`0x44` GPIO6) | Yes | I²C write |
| 2× PI4IOE5V6408 (`0x43` for BtnA/B, `0x44` for BL + other) | Yes | Wire + PI4IO driver |
| Piezo buzzer | Yes, GPIO11 | ledcWrite |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / SCLK | 21 / 20 |
| LCD CS / DC | 17 / 16 |
| LCD RST | — (not a GPIO; handled via PI4IOE5V6408 #2) |
| Internal I²C SCL / SDA | 8 / 10 (both expanders) |
| Port B / C pin1/pin2 | 4 / 5 |
| Buzzer | 11 |
| I/O expander #1 (BtnA/B) | I²C `0x43` |
| I/O expander #2 (BL + LCD control) | I²C `0x44` |

---

## 1. Display (backlight via I²C)

`M5.Display.setBrightness()` ends up calling `pi4io_write(0x44, 0x05, bit_on)` — set GPIO6 high on expander #2.

Standalone LovyanGFX setup: use the ST7789 config from [M5GFX.cpp:2341](../../M5GFX/src/M5GFX.cpp#L2341) (autodetect branch for Nesso).

Backlight on manually:
```cpp
Wire.begin(10, 8, 400000);
Wire.beginTransmission(0x44);
Wire.write(0x03); Wire.write(0b01000000);   // GPIO6 = output
Wire.endTransmission();
Wire.beginTransmission(0x44);
Wire.write(0x05); Wire.write(0b01000000);   // GPIO6 = HIGH (BL on)
Wire.endTransmission();
```

---

## 2. Buttons

| Button | Source |
|--------|--------|
| BtnA | expander `0x43`, reg `0x0F`, bit 0 (inverted) |
| BtnB | expander `0x43`, reg `0x0F`, bit 1 (inverted) |

```cpp
uint8_t state = pi4io_read(0x43, 0x0F);
bool btnA = !(state & 0x01);
bool btnB = !(state & 0x02);
```

See [M5UnitC6L.md](M5UnitC6L.md#2-io-expander-pi4ioe5v6408--0x43) for the `pi4io_read` helper.

---

## 3. Buzzer

GPIO11 PWM buzzer (same as UnitC6L).

---

## Brightness (LCD backlight — on/off only)

Same pattern as StampPLC but **different expander and active-HIGH**:

```cpp
void nesson1_bl(uint8_t b) {
  if (b) pi4io_bit_on (0x44, 0x05, 1 << 6);    // GPIO6 HIGH = BL on
  else   pi4io_bit_off(0x44, 0x05, 1 << 6);
}
```

`setBrightness()` is effectively binary — 0 = off, anything else = on. No PWM dimming.

## Volume (buzzer)

GPIO11 buzzer, `magnification = 48`. Standard software path.

---

## Power management

USB-powered only — no battery. Nesso's PI4IOE5V6408 #2 does gate some auxiliary rails, but these are static — no runtime power management API.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Nesso autodetect | [M5GFX.cpp:2201](../../M5GFX/src/M5GFX.cpp#L2201) |
| Nesso pinmap | [src/M5Unified.cpp:2580](src/M5Unified.cpp#L2580) |
| PI4IOE5V6408 driver | [src/utility/PI4IOE5V6408_Class.cpp](src/utility/PI4IOE5V6408_Class.cpp) |
