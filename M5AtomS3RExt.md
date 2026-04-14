# M5AtomS3RExt — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomS3RExt` (143)
**SoC:** ESP32-S3

AtomS3R **without** the LCD — LCD pins (GPIO 21, 15, 14, 42, 48) are exposed on a header instead. Keeps BMI270 + BMM150 9-axis IMU.

---

## Differences from [M5AtomS3R](M5AtomS3R.md)

| Item | AtomS3R | AtomS3RExt |
|------|---------|------------|
| ST7735/GC9107 LCD | ✅ | **removed** |
| I²C backlight IC (`0x30`) | ✅ | **removed** |
| GPIO 21, 15, 14, 42, 48 | LCD bus | **free user header** |
| BMI270 + BMM150 IMU | ✅ | ✅ |

All other sections (internal I²C pins, BtnA, Grove) are identical.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Internal I²C SCL / SDA** | **0 / 45** (IMU + mag) |
| Grove (Port A) SCL / SDA | 1 / 2 |
| BtnA | 41 |
| Header pins (free to use) | 14, 15, 21, 42, 48 |

---

## IMU

See [M5AtomS3R.md § BMI270 IMU](M5AtomS3R.md#2-bmi270-imu--bmm150-mag). The axis fix-up and driver code are identical.

---

## Minimal sketch

```cpp
#include <Wire.h>
void setup() {
  Wire1.begin(45, 0, 400000);
  pinMode(41, INPUT_PULLUP);
  // init BMI270 + BMM150 per Bosch API
}
```

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomS3RExt pinmap | [src/M5Unified.cpp:1659](src/M5Unified.cpp#L1659) |
