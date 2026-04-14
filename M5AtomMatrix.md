# M5AtomMatrix — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomMatrix` (141)
**SoC:** ESP32-PICO-D4

AtomLite + 5×5 WS2812 matrix on the face + MPU6886 IMU.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 matrix (25 pixels, GPIO27) | Yes | Adafruit_NeoPixel or FastLED |
| MPU6886 IMU (6-axis) | Yes, I²C `0x68` on internal bus | Adafruit MPU6886 or raw Wire |
| BtnA (matrix is pressable) | Yes, GPIO39 | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |
| Internal I²C bus | Yes (shared with IMU) | Arduino Wire1 |
| Display / Speaker / Mic / RTC | **No** | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 matrix data | 27 |
| BtnA | 39 |
| Internal I²C SCL / SDA | 21 / 25 |
| Grove (Port A) SCL / SDA | 32 / 26 |

Same GPIO0 WiFi-noise fix as AtomLite: `digitalWrite(0, HIGH)` at boot.

---

## 1. LED matrix (25 pixels)

Layout is **row-major**, top-left = `(x=0, y=0)` = index 0, bottom-right = index 24.

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel matrix(25, 27, NEO_GRB + NEO_KHZ800);

int xy(int x, int y) { return y * 5 + x; }

void setup() {
  matrix.begin();
  matrix.setBrightness(32);     // keep low; 25 pixels at full white = ~1.5 A
  matrix.setPixelColor(xy(2, 2), matrix.Color(0, 255, 0)); // center green
  matrix.show();
}
```

---

## 2. MPU6886 IMU (I²C `0x68`)

The MPU6886 is on the internal I²C bus (SCL=21, SDA=25). M5Unified's axis fix-up inverts **X and Z** (both accel and gyro) — if you query the chip directly you must do the same for readings to match Matrix orientation.

```cpp
#include <Wire.h>

#define IMU_ADDR 0x68
void imu_write(uint8_t reg, uint8_t val) {
  Wire1.beginTransmission(IMU_ADDR);
  Wire1.write(reg); Wire1.write(val);
  Wire1.endTransmission();
}
int16_t imu_read16(uint8_t reg) {
  Wire1.beginTransmission(IMU_ADDR);
  Wire1.write(reg);
  Wire1.endTransmission(false);
  Wire1.requestFrom(IMU_ADDR, 2);
  int16_t v = (Wire1.read() << 8) | Wire1.read();
  return v;
}

void setup() {
  Wire1.begin(25, 21, 400000);
  imu_write(0x6B, 0x00);   // PWR_MGMT_1: wake
  imu_write(0x1C, 0x00);   // ACCEL_CONFIG: ±2g
  imu_write(0x1B, 0x00);   // GYRO_CONFIG: ±250 dps
}

void loop() {
  int16_t ax = imu_read16(0x3B);
  int16_t ay = imu_read16(0x3D);
  int16_t az = imu_read16(0x3F);
  // Matrix orientation fix:
  ax = -ax;  az = -az;
  Serial.printf("accel: %d %d %d\n", ax, ay, az);
  delay(100);
}
```

For the full register table + calibration, see [src/utility/imu/MPU6886_Class.cpp](src/utility/imu/MPU6886_Class.cpp).

---

## 3. Button + Grove port

Same as AtomLite: `pinMode(39, INPUT_PULLUP)`; `Wire.begin(26, 32, 400000)`.

---

## Power management

USB-powered only — see [M5AtomLite.md § Power management](M5AtomLite.md#power-management). Watch total current draw: 25 WS2812s at full white ≈ 1.5 A, exceeds most USB-C sources.

---

## Source references

| Topic | File : line |
|-------|-------------|
| MPU6886 driver | [src/utility/imu/MPU6886_Class.cpp](src/utility/imu/MPU6886_Class.cpp) |
| Matrix axis fix-up | [src/utility/IMU_Class.cpp](src/utility/IMU_Class.cpp) — search `AtomMatrix` |
| WS2812 RMT | [src/utility/led/LED_Strip_Class.cpp](src/utility/led/LED_Strip_Class.cpp) |
