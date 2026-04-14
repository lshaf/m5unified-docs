# M5VAMeter — standalone code

**Board enum:** `board_M5VAMeter` (16)
**SoC:** ESP32-S3

Round 240×240 ST7789 display + INA226 current/voltage sensor. No battery, no RTC.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 round LCD (240×240) | Yes | M5GFX / LovyanGFX `Panel_ST7789` |
| INA226 current sensor | Yes, I²C `0x40` on internal bus | `INA226_WE` or M5Unified's class |
| Piezo buzzer | Yes, GPIO14 | ledcWrite |
| BtnA / BtnB | Yes | Arduino digitalRead |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / SCLK | 35 / 36 |
| LCD CS / DC / RST | 37 / 34 / 33 |
| LCD BL | 38 (LEDC ch7 PWM @512 Hz) |
| Internal I²C SCL / SDA | 6 / 5 (INA226) |
| Grove (Port A) SCL / SDA | 9 / 8 |
| Buzzer | 14 |
| BtnA | 2 |
| BtnB | 0 |

---

## 1. Display

M5GFX autodetect: [M5GFX.cpp:1698](../../M5GFX/src/M5GFX.cpp#L1698) (shares the Cardputer/CardputerADV autodetect branch; VAMeter is the 240×240 panel variant, detected by panel ID).

LovyanGFX `Panel_ST7789` config (for standalone use):
- MOSI=35, SCLK=36, CS=37, DC=34, RST=33
- panel_width = panel_height = 240
- offset_x=0, offset_y=0
- backlight: pin 38, PWM channel 7, 512 Hz

---

## 2. INA226 sensor

```cpp
#include <Wire.h>
#define INA_ADDR 0x40
void setup() {
  Wire1.begin(5 /* SDA */, 6 /* SCL */, 400000);
}
uint16_t ina_read(uint8_t reg) {
  Wire1.beginTransmission(INA_ADDR);
  Wire1.write(reg);
  Wire1.endTransmission(false);
  Wire1.requestFrom(INA_ADDR, 2);
  return (Wire1.read() << 8) | Wire1.read();
}
// Read bus voltage (reg 0x02) — LSB = 1.25 mV
float vbus = ina_read(0x02) * 0.00125f;
```

Full driver: [src/utility/power/INA226_Class.cpp](src/utility/power/INA226_Class.cpp).

---

## 3. Buzzer / Buttons

GPIO14 buzzer with `ledcWrite`. Buttons on GPIO2 / GPIO0 with `INPUT_PULLUP`.

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO38, LEDC ch7, 512 Hz, 9-bit**.

```cpp
void bl_init() { ledcSetup(7, 512, 9); ledcAttachPin(38, 7); }
void bl_set(uint8_t b) { ledcWrite(7, b << 1); }
```

## Volume (buzzer)

GPIO14 buzzer, `magnification = 48`. Standard buzzer volume path.

---

## Power management

VAMeter has **no battery** — it's mains/USB-powered through the device under test's power rails. `M5.Power.getBatteryLevel()` returns `-2` (unsupported). What's useful:

```cpp
float v = M5.Power.Ina226.getBusVoltage();        // measured bus voltage (volts)
float i = M5.Power.Ina226.getCurrent();           // amps
float p = M5.Power.Ina226.getPower();             // watts
```

INA226 configuration (already done by M5Unified): shunt 0.01 Ω, max 2 A. Change via `Ina226.config(...)`.

Standalone INA226 read (addr `0x40` on internal bus 6/5):
```cpp
uint16_t ina226_read(uint8_t reg) {
  Wire1.beginTransmission(0x40); Wire1.write(reg); Wire1.endTransmission(false);
  Wire1.requestFrom(0x40, 2);
  return (Wire1.read() << 8) | Wire1.read();
}
float bus_voltage()  { return ina226_read(0x02) * 0.00125f; }  // V, 1.25 mV/LSB
float shunt_voltage(){ return (int16_t)ina226_read(0x01) * 0.0000025f; } // V
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| VAMeter autodetect (shared branch) | [M5GFX.cpp:1647](../../M5GFX/src/M5GFX.cpp#L1647) |
| VAMeter pinmap | [src/M5Unified.cpp:1668](src/M5Unified.cpp#L1668) |
| INA226 driver | [src/utility/power/INA226_Class.cpp](src/utility/power/INA226_Class.cpp) |
