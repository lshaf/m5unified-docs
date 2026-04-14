# M5StampPLC — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StampPLC` (21)
**SoC:** ESP32-S3

PLC-style industrial controller: ST7789 135×240 LCD + RX8130 RTC (battery-backed) + PI4IOE5V6408 I/O expander for isolated inputs/outputs + SD.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD | Yes, SPI | M5GFX |
| **RX8130 RTC (not PCF8563!)** | Yes, I²C `0x32` | driver in [src/utility/rtc/RX8130_Class.cpp](src/utility/rtc/RX8130_Class.cpp) |
| PI4IOE5V6408 I/O expander | Yes, I²C `0x43` | Wire |
| Piezo buzzer | Yes, GPIO44 | ledcWrite |
| MicroSD SPI | Yes | `SD.h` |
| WS2812 | Yes, GPIO21 | Adafruit_NeoPixel |
| Battery | **No** (mains-powered, but RTC has coin cell) | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / MISO / SCLK | 8 / 9 / 7 (**shared with SD!**) |
| LCD CS / DC / RST | 12 / 6 / 3 |
| LCD BL | **via PI4IOE5V6408 GPIO7** (I²C control, not a direct GPIO) |
| SD SPI SCLK / MOSI / MISO / CS | 7 / 8 / 9 / 10 |
| Internal I²C SCL / SDA | 15 / 13 |
| Grove (Port A) SCL / SDA | 1 / 2 |
| Buzzer | 44 |
| WS2812 | 21 |
| BtnA / BtnB / BtnC | expander `0x0F` bits 2 / 1 / 0 (all inverted) |

### Bus-sharing: SD + LCD share SPI

LCD and SD share the same SPI bus (pins 7/8/9). M5GFX handles CS arbitration. If using both, call `M5GFX::init()` before `SD.begin()`.

---

## 1. Display

Autodetect: [M5GFX.cpp:1803](../../M5GFX/src/M5GFX.cpp#L1803). Backlight is turned on by writing PI4IO (`0x43`) reg `0x05` bit for GPIO7.

---

## 2. RX8130 RTC (different from PCF8563)

```cpp
#include <Wire.h>
#define RX_ADDR 0x32
void setup() {
  Wire1.begin(13, 15, 400000);
}
// Read time: 7 bytes starting at reg 0x10 (sec, min, hour, week, day, month, year BCD)
Wire1.beginTransmission(RX_ADDR); Wire1.write(0x10); Wire1.endTransmission(false);
Wire1.requestFrom(RX_ADDR, 7);
```

Full driver: [src/utility/rtc/RX8130_Class.cpp](src/utility/rtc/RX8130_Class.cpp). Registers are **different from PCF8563** — do not use the PCF8563 driver for this board.

---

## 3. I/O expander for buttons

```cpp
uint8_t v = pi4io_read(0x43, 0x0F);
bool btnA = !(v & 0b100);
bool btnB = !(v & 0b010);
bool btnC = !(v & 0b001);
```

---

## 4. Buzzer / LED

GPIO44 buzzer, GPIO21 WS2812.

---

## Brightness (LCD backlight — on/off only)

**Not PWM** — a single PI4IOE5V6408 GPIO pin toggles the backlight. Active-LOW.

```cpp
// one-time init (set GPIO7 as output, push-pull, no pull):
pi4io_bit_on (0x43, 0x03, 1 << 7);
pi4io_bit_off(0x43, 0x0D, 1 << 7);
pi4io_bit_off(0x43, 0x07, 1 << 7);

void stampplc_bl(uint8_t b) {
  if (b == 0) pi4io_bit_on (0x43, 0x05, 1 << 7);   // high = BL off
  else        pi4io_bit_off(0x43, 0x05, 1 << 7);   // low  = BL on
}
```

`lcd.setBrightness(1..255)` → all map to "on"; only `setBrightness(0)` turns it off. **No dimming** possible.

## Volume (buzzer)

GPIO44 buzzer, `magnification = 48`. Software duty path only.

---

## Power management

Mains-powered (12/24 V DC input rail). No battery. StampPLC has an **INA226** on internal bus for measuring supply current — useful for monitoring PLC load:

```cpp
float v_in = M5.Power.Ina226.getBusVoltage();    // V
float i_in = M5.Power.Ina226.getCurrent();       // A
float p_in = M5.Power.Ina226.getPower();         // W
```

INA226 is configured with a 0.01 Ω shunt, 2 A max (set in M5Unified).

Coin-cell on the RX8130 keeps the clock alive across power loss.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StampPLC autodetect | [M5GFX.cpp:1803](../../M5GFX/src/M5GFX.cpp#L1803) |
| StampPLC pinmap | [src/M5Unified.cpp:1685](src/M5Unified.cpp#L1685), [:2505](src/M5Unified.cpp#L2505) |
| RX8130 driver | [src/utility/rtc/RX8130_Class.cpp](src/utility/rtc/RX8130_Class.cpp) |
| PI4IOE5V6408 driver | [src/utility/PI4IOE5V6408_Class.cpp](src/utility/PI4IOE5V6408_Class.cpp) |
