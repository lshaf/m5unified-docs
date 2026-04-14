# M5PowerHub — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5PowerHub` (146)
**SoC:** ESP32-S3

USB-PD / power-distribution hub. Uses a **dedicated co-MCU** at I²C `0x50` that exposes LED, buttons, RTC, and per-port power metrics. Multiple INA3221 current sensors are also on the bus.

No LCD. Most "peripherals" are behind the co-MCU, not direct GPIO.

---

## What's onboard

| Component | Accessed via | Standalone library |
|-----------|--------------|--------------------|
| Co-MCU (buttons + LED + RTC + rails) | I²C `0x50` | raw Wire + register map |
| INA3221 sensors (per-port V/I) | I²C | Beastie/INA3221 or similar |
| BtnA | direct GPIO11 | digitalRead |
| BtnB | Co-MCU reg `0xA0` bit 0 | I²C read |

No display, no IMU, no audio, no PCF8563 RTC (co-MCU has its own).

---

## Pin map

| Purpose | GPIO |
|---------|------|
| Internal I²C SCL / SDA | 48 / 45 |
| Grove (Port A) SCL / SDA | 16 / 15 |
| Port C pin1 / pin2 | 1 / 2 |
| BtnA | 11 (direct) |

---

## 1. Reading buttons

```cpp
#include <Wire.h>
void setup() {
  Wire1.begin(45, 48, 400000);
  pinMode(11, INPUT_PULLUP);
}
bool btnA() { return digitalRead(11) == LOW; }
bool btnB() {
  Wire1.beginTransmission(0x50);
  Wire1.write(0xA0); Wire1.endTransmission(false);
  Wire1.requestFrom(0x50, 1);
  uint8_t v = Wire1.read();
  return !(v & 0x01);
}
```

---

## 2. LED via co-MCU

M5Unified uses `LED_PowerHub_Class` ([src/utility/led/LED_PowerHub_Class.cpp](src/utility/led/LED_PowerHub_Class.cpp)). Writes are routed to co-MCU over I²C — not RMT. Copy that class if you want to reproduce it standalone.

---

## 3. RTC via co-MCU

`RTC_PowerHub_Class` ([src/utility/rtc/RTC_PowerHub_Class.cpp](src/utility/rtc/RTC_PowerHub_Class.cpp)) reads/writes the co-MCU's RTC registers. Different register set from PCF8563 / RX8130.

---

## 4. Per-port power monitoring

Each USB-PD output port has an INA3221. Addresses depend on per-unit strap — probe I²C bus to find them:

```cpp
for (uint8_t a = 0x40; a <= 0x43; ++a) {
  Wire1.beginTransmission(a);
  if (Wire1.endTransmission() == 0) Serial.printf("INA3221 at 0x%02X\n", a);
}
```

Driver: [src/utility/power/INA3221_Class.cpp](src/utility/power/INA3221_Class.cpp).

---

## Power management

PowerHub is itself a PMIC — the co-MCU at `0x50` manages USB-PD negotiation, per-port switching, and battery. `M5.Power.*` maps to the co-MCU register set.

### With M5Unified

```cpp
int pct  = M5.Power.getBatteryLevel();       // 0..100
int mv   = M5.Power.getBatteryVoltage();     // reads getBatteryVoltage/2 path
auto chg = M5.Power.isCharging();             // (getBatteryCurrent() < -10)
int32_t cur = M5.Power.getBatteryCurrent();   // signed mA
```

### Standalone — per-port voltage/current via INA3221

Multiple INA3221s sit on the internal bus. M5Unified scans addresses `0x40`/`0x41`. Per-port register mapping is in [src/utility/Power_Class.cpp:1795](../src/utility/Power_Class.cpp#L1795) (a `PortReg` table matching `ext_port_mask_t` values to co-MCU registers).

Practical: use the M5Unified API unless you're building a specific PowerHub tool.

### Battery

The battery is managed entirely by the co-MCU. You cannot change charge current/voltage — those are preset.

---

## Source references

| Topic | File : line |
|-------|-------------|
| PowerHub pinmap | [src/M5Unified.cpp:1703](src/M5Unified.cpp#L1703), [:2516](src/M5Unified.cpp#L2516) |
| LED driver | [src/utility/led/LED_PowerHub_Class.cpp](src/utility/led/LED_PowerHub_Class.cpp) |
| RTC driver | [src/utility/rtc/RTC_PowerHub_Class.cpp](src/utility/rtc/RTC_PowerHub_Class.cpp) |
| INA3221 driver | [src/utility/power/INA3221_Class.cpp](src/utility/power/INA3221_Class.cpp) |
