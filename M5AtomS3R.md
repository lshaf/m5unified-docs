# M5AtomS3R — standalone code

**Board enum:** `board_M5AtomS3R` (18)
**SoC:** ESP32-S3 (8 MB PSRAM + 8 MB Flash)

AtomS3 + **BMI270 6-axis IMU + BMM150 magnetometer (9-axis fusion possible)** and an I²C-controlled backlight IC.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7735 or GC9107 128×128 LCD (auto-detected) | Yes | M5GFX |
| I²C backlight IC (addr `0x30`) | Yes | M5GFX's `Light_M5StackAtomS3R` OR raw Wire |
| BMI270 IMU (accel + gyro) | Yes, I²C `0x68` or `0x69` | `Bosch-Sensortec/BMI270_SensorAPI` |
| BMM150 magnetometer | Yes, I²C `0x10` | `BoschSensortec/BMM150-Sensor-API` |
| BtnA | Yes, GPIO41 | digitalRead |
| Grove Port A | Yes | Wire |

---

## Pin map

| Purpose | GPIO | Notes |
|---------|------|-------|
| LCD MOSI / SCLK / CS / DC / RST | 21 / 15 / 14 / 42 / 48 | SPI |
| **Internal I²C SCL / SDA** | **0 / 45** | BMI270, BMM150, BL IC |
| Grove (Port A) SCL / SDA | 1 / 2 | |
| BtnA | 41 | |

⚠️ Internal I²C uses GPIO0 for SCL — this is also the boot strapping pin. During normal operation it's fine, but it must be HIGH at reset (which I²C idle already guarantees).

---

## 1. Display (autodetect)

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  lcd.init();
  lcd.setBrightness(128);   // writes via I²C to backlight IC `0x30`
}
```

Autodetect: [M5GFX.cpp:1857](../../M5GFX/src/M5GFX.cpp#L1857).

### Backlight directly (skip M5GFX)

The backlight is **not a GPIO PWM** — it's an I²C chip at `0x30` on the internal bus. Brightness is set via a register write:

```cpp
// uses the Light_M5StackAtomS3R class from M5GFX.cpp:450
Wire1.begin(45 /* SDA */, 0 /* SCL */, 400000);
uint8_t bl_value = 0x80;     // 0..0xFF brightness
Wire1.beginTransmission(0x30);
Wire1.write(0x00); Wire1.write(bl_value);
Wire1.endTransmission();
```

Full class: [M5GFX.cpp:450](../../M5GFX/src/M5GFX.cpp#L450).

---

## 2. BMI270 IMU + BMM150 mag

Both sit on internal I²C (SDA=45, SCL=0). M5Unified auto-detects which ID the BMI270 responds to (`0x68` vs `0x69`) and rewires axis order to match the physical orientation of the Atom.

Axis fix-up rules (applied in [src/utility/IMU_Class.cpp](src/utility/IMU_Class.cpp)):
- **Accel + gyro:** reorder Y→X, X→Y, keep Z, **invert new Y**
- **Mag (BMM150):** invert X and Z

### Standalone BMI270 init (minimal)

```cpp
#include <Wire.h>
#define BMI_ADDR 0x68   // try 0x68, fall back to 0x69

void setup() {
  Wire1.begin(45, 0, 400000);
  // BMI270 requires a config-file upload (see Bosch's BMI270 init blob ~8KB)
  // Easiest: use the official BMI270_SensorAPI library, which bundles the blob.
}
```

The config blob is ~8 KB and must be written via register `0x5E` burst write. Full sequence: [src/utility/imu/BMI270_Class.cpp](src/utility/imu/BMI270_Class.cpp) + `BMI270_config.inl` (blob included).

### BMM150 (simpler)

```cpp
// Standard Bosch API, I²C addr 0x10
// Power mode 0x4B set normal, then read 6 bytes from 0x42 for raw XYZ.
```
Driver: [src/utility/imu/BMM150_Class.cpp](src/utility/imu/BMM150_Class.cpp).

---

## 3. Grove / Button

Same as AtomS3 — `pinMode(41, INPUT_PULLUP)`, `Wire.begin(2, 1, 400000)`.

---

## Minimal sketch

```cpp
#include <M5GFX.h>
#include <M5Unified.h>       // only for IMU_Class headers
M5GFX lcd;
m5::IMU_Class imu;

void setup() {
  lcd.init(); lcd.setBrightness(128);
  // IMU_Class::begin() needs an I2C_Class; easiest is to piggy-back on M5 object:
  imu.begin(&m5::In_I2C);
}

void loop() {
  m5::IMU_Class::imu_data_t d;
  imu.getImuData(&d);
  lcd.setCursor(0, 0);
  lcd.printf("ax=%.2f\nay=%.2f\naz=%.2f\n", d.accel.x, d.accel.y, d.accel.z);
  delay(100);
}
```

---

## Brightness (LCD backlight)

Unique: **dedicated I²C backlight IC at `0x30`** on internal bus (SDA=45, SCL=0).

One-time init:
```cpp
Wire1.begin(45, 0, 400000);
// IC datasheet-specific init (from M5GFX Light_M5StackAtomS3R):
Wire1.beginTransmission(0x30); Wire1.write(0x00); Wire1.write(0b01000000); Wire1.endTransmission(); delay(1);
Wire1.beginTransmission(0x30); Wire1.write(0x08); Wire1.write(0b00000001); Wire1.endTransmission();
Wire1.beginTransmission(0x30); Wire1.write(0x70); Wire1.write(0b00000000); Wire1.endTransmission();
```

Set brightness — **direct 0..255 byte to register `0x0E`**:
```cpp
void atoms3r_bl(uint8_t b) {
  Wire1.beginTransmission(0x30);
  Wire1.write(0x0E); Wire1.write(b);
  Wire1.endTransmission();
}
```

With M5GFX: `lcd.setBrightness(0..255)` — it runs the init sequence plus the register write for you.

## Volume

No onboard speaker.

---

## Power management

USB-powered only — no battery. `M5.Power.getBatteryLevel()` returns `-2`.

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomS3R autodetect (panel + BL IC) | [M5GFX.cpp:1857](../../M5GFX/src/M5GFX.cpp#L1857) |
| AtomS3R pinmap | [src/M5Unified.cpp:1658](src/M5Unified.cpp#L1658) |
| BMI270 driver + config blob | [src/utility/imu/BMI270_Class.cpp](src/utility/imu/BMI270_Class.cpp), [BMI270_config.inl](src/utility/imu/BMI270_config.inl) |
| BMM150 mag driver | [src/utility/imu/BMM150_Class.cpp](src/utility/imu/BMM150_Class.cpp) |
| Axis fix-up (for AtomS3R) | [src/utility/IMU_Class.cpp](src/utility/IMU_Class.cpp) — search `board_M5AtomS3R` |
