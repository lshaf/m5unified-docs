# M5Paper — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5Paper` (7)
**SoC:** ESP32-D0WDQ6 + 8 MB PSRAM

4.7" e-paper tablet with **IT8951 controller** + **GT911 capacitive touch**.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| IT8951 e-paper controller (960×540) | Yes | M5GFX OR GxEPD2_IT8951 |
| GT911 cap-touch | Yes, I²C `0x5D` | GT911 libraries |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| MicroSD SPI | Yes | `SD.h` |
| Battery (simple charger) | Yes | ADC on GPIO35 |
| Power hold | GPIO2 | digitalWrite |
| WS2812 LED | Yes, GPIO4 (shared with SD CS — careful!) | Adafruit_NeoPixel |

No speaker, no mic, no IMU, no PMIC.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold** | **2** (HIGH = stay on) |
| IT8951 MOSI / MISO / SCLK | 12 / 13 / 14 |
| IT8951 CS / RST / BUSY | 15 / 23 / 27 |
| Internal I²C SCL / SDA | 22 / 21 (RTC) |
| Grove (Port A) SCL / SDA | 32 / 25 |
| Port B pin1/pin2 | 33 / 26 |
| Port C pin1/pin2 | 19 / 18 |
| SD SPI SCLK/MOSI/MISO/CS | 14 / 12 / 13 / 4 |
| Touch I²C | shared with Port A (32/25) OR internal I²C? — check GT911 schematic |
| Touch INT / RST | GPIO36 / 23 (⚠️ RST shared with IT8951 RST — toggle carefully) |
| Battery ADC | 35 |
| BtnA / BtnB / BtnC | 37 / 38 / 39 |

### ⚠️ GPIO23 is shared between IT8951 RST and GT911 RST

If you toggle GT911 reset without care, you also reset the IT8951 panel. Order of bring-up matters — reset IT8951 first, then GT911.

### ⚠️ GPIO4 WS2812 vs SD CS

Without M5Unified you must decide which you're using — LED OR SD — not both simultaneously on the same pin.

---

## 1. Power hold

```cpp
void setup() {
  pinMode(2, OUTPUT);
  digitalWrite(2, HIGH);    // LATCH on battery
}
```

---

## 2. E-paper display

```cpp
#include <M5GFX.h>
M5GFX epd;
void setup() {
  digitalWrite(2, HIGH); pinMode(2, OUTPUT);
  epd.init();
  epd.setEpdMode(epd_mode_t::epd_fast);   // or epd_quality for 16-gray
}
```

M5GFX handles the IT8951 init (SYS_RUN/STANDBY commands, waveform table load). If writing your own driver, follow Waveshare's IT8951 demo sequence — the chip expects `SYS_RUN` on startup, then `LOAD_IMG_AREA` + `DPY_AREA`. M5GFX source: [Panel_IT8951.hpp](../../M5GFX/src/lgfx/v1/panel/Panel_IT8951.hpp).

---

## 3. Touch (GT911 `0x5D`)

```cpp
Wire.begin(25, 32, 400000);
Wire.beginTransmission(0x5D);
Wire.write(0x81); Wire.write(0x4E);        // status register
Wire.endTransmission(false);
Wire.requestFrom(0x5D, 1);
uint8_t status = Wire.read();
// if status bit 7 set: new touch data available; read N*8 bytes from 0x8150
```

Use a GT911 Arduino library (e.g. `TAMC_GT911`) for simplicity.

---

## 4. RTC / SD / Battery

- RTC: PCF8563 at `0x51` on internal I²C (21/22).
- SD: `SD.begin(4, SPI, 20000000)` — but watch CS=4 collision with WS2812.
- Battery: `analogRead(35)` through the ADC calibration curve (see M5Unified Power class for the exact mV formula).

---

## 5. Buttons

GPIO37/38/39, `INPUT_PULLUP`.

---

## Brightness / Volume

E-paper — **no backlight**, no onboard speaker. `setBrightness()` is a no-op. With SPK HAT see [M5StickCPlus.md § Volume](M5StickCPlus.md#volume-buzzer).

---

## Power management

Simple charger + ADC battery read — no PMIC.

### With M5Unified

```cpp
int pct = M5.Power.getBatteryLevel();     // 0..100
int mv  = M5.Power.getBatteryVoltage();   // mV
auto chg = M5.Power.isCharging();          // charge_unknown
```

### Standalone battery voltage (ADC on GPIO35, 2:1 divider)

```cpp
float vbat_mv() {
  int raw = analogRead(35);
  return (raw * 3300.0f / 4095.0f) * 2.0f;
}

int pct_from_mv(float mv) {
  int pct = (mv - 3300) * 100 / (4150 - 3350);
  return (pct < 0) ? 0 : (pct > 100) ? 100 : pct;
}
```

### Power-off / shutdown

Pull **GPIO2 LOW** to cut the latched 3.3 V rail. M5Unified does this in `Power.powerOff()`.

No dedicated charging-status GPIO — use voltage-trend detection if you need "is_charging" status standalone.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Paper autodetect | [M5GFX.cpp:1139](../../M5GFX/src/M5GFX.cpp#L1139) |
| Paper pinmap | [src/M5Unified.cpp:1606](src/M5Unified.cpp#L1606) |
| IT8951 panel driver | [M5GFX/src/lgfx/v1/panel/Panel_IT8951.hpp](../../M5GFX/src/lgfx/v1/panel/Panel_IT8951.hpp) |
