# M5StickC — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StickC` (3)
**SoC:** ESP32-PICO-D4

Original 80×160 color LCD Stick with AXP192 PMIC.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7735 LCD 80×160 | Yes | M5GFX |
| PDM microphone (SPM1423) | Yes, via AXP192 LDO0 | ESP-IDF `i2s_pdm_rx` |
| AXP192 PMIC | Yes, I²C `0x34` | AXP192 driver |
| MPU6886 IMU (or SH200Q) | Yes | MPU6886/SH200Q driver |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| Red onboard LED | Yes, GPIO10 (active LOW) | digitalWrite |
| IR TX LED | Yes, GPIO9 | IRremote or ledcWrite |
| BtnA / BtnB | Yes, GPIO37 / 39 | digitalRead |

No speaker (HAT-only), no SD card, no touch.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / MISO / SCLK | 15 / 14 / 13 |
| LCD CS / DC / RST | 5 / 23 / 18 |
| **LCD backlight** | **AXP192 LDO2** (no GPIO — set LDO2 to ~3.3 V) |
| PDM DATA-IN | 34 |
| PDM CLK (`pin_ws`) | 0 |
| Internal I²C SCL / SDA | 22 / 21 (PMIC + IMU + RTC) |
| Grove (Port A) SCL / SDA | 33 / 32 |
| Red LED | 10 |
| IR TX | 9 |
| BtnA / BtnB | 37 / 39 |

### Boot-time GPIO0 fix

M5Unified forces `digitalWrite(0, HIGH)` at boot to eliminate CH552 USB-UART ↔ WiFi interference. Replicate if you skip M5Unified. PDM mic CLK shares GPIO0, so this is only done BEFORE the mic is configured.

---

## ⚠️ LCD needs AXP192 LDO2 to power up

Just like Core2, the ST7735 panel has no direct power — both the logic rail AND the backlight come from a single AXP192 rail (LDO2). If LDO2 is 0, the screen is black.

```cpp
void axp192_write(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(0x34); Wire.write(reg); Wire.write(val); Wire.endTransmission();
}
void setup() {
  Wire.begin(21, 22, 400000);
  // LDO2 = 3.3V:
  uint8_t ldo23 = (0b11111 << 4) | 0b1111;  // LDO2=3.3V, LDO3=3.3V
  axp192_write(0x28, ldo23);
  axp192_write(0x12, 0x04);                 // enable LDO2 (bit 2)
}
```

Or use M5GFX which handles this automatically.

---

## 1. Display

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() { lcd.init(); lcd.setBrightness(128); }
```

Autodetect: [M5GFX.cpp:738](../../M5GFX/src/M5GFX.cpp#L738).

Explicit `Panel_ST7735S` config if skipping M5GFX:
- Bus: MOSI=15, SCLK=13, DC=23 (MISO=14 if readable)
- Panel: CS=5, RST=18, width=80, height=160, offset_x=26, offset_y=1, invert=true
- Backlight: **via AXP192 LDO2** (not a GPIO)

---

## 2. PDM microphone

Requires AXP192 LDO0 = 2.8 V before the mic will respond (driver source: [src/M5Unified.cpp:605](src/M5Unified.cpp#L605)).

```cpp
// enable mic power:
uint8_t ldo0 = (0b1010 << 4) | 0b0010;  // LDO0 = 2.8 V
axp192_write(0x91, ldo0);
axp192_write(0x90, axp192_read(0x90) | 0x02);  // enable LDO0 in GPIO0 mode
// then i2s_pdm_rx on DIN=34, CLK=0
```

Disable = `axp192_write(0x90, xxx & ~0x02)`.

---

## 3. IMU (MPU6886 or SH200Q, auto-detect)

Probe address `0x68` for MPU6886, fall back to `0x6C` for SH200Q.

```cpp
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x68); Wire.write(0x75); Wire.endTransmission(false);
Wire.requestFrom(0x68, 1);
uint8_t who = Wire.read();       // 0x19 = MPU6886
```

Driver: [src/utility/imu/MPU6886_Class.cpp](src/utility/imu/MPU6886_Class.cpp) / [SH200Q_Class.cpp](src/utility/imu/SH200Q_Class.cpp).

---

## 4. RTC

PCF8563 at `0x51` on internal bus. See [PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp).

---

## 5. Buttons

```cpp
pinMode(37, INPUT_PULLUP);  // BtnA (front)
pinMode(39, INPUT_PULLUP);  // BtnB (side)
```

---

## Brightness (LCD backlight)

**AXP192 LDO2** voltage (same rail as LCD logic). Register 0x28 HIGH nibble. ~13 discrete steps.

```cpp
void stickc_brightness(uint8_t b) {
  if (!b) { axp192_mod(0x12, 0, 0x04); return; }       // LDO2 disable → screen off
  uint8_t v = (((b >> 1) + 8) / 13) + 5;                // 5..15 → 2.5..3.3 V
  axp192_mod(0x12, 0x04, 0x04);                         // LDO2 enable
  axp192_mod(0x28, v << 4, 0xF0);                       // high nibble = LDO2 voltage
}
```

**Note:** below `b=1` the voltage drops to ~2.5 V and the panel barely renders. There's no "true off" that keeps logic alive — LDO2 feeds both logic and backlight.

## Volume

M5StickC has no onboard speaker — only the mic and HAT-speaker option. For HAT SPK / SPK2 volume see [M5StickCPlus.md](M5StickCPlus.md#volume-buzzer).

---

## Power management

PMIC: **AXP192** @ I²C `0x34`.

### With M5Unified

```cpp
int   level = M5.Power.getBatteryLevel();          // 0..100 %
int   mv    = M5.Power.getBatteryVoltage();        // mV
auto  chg   = M5.Power.isCharging();               // is_charging / is_discharging / charge_unknown
int   vbus  = M5.Power.getVBUSVoltage();           // USB bus mV
int32_t cur = M5.Power.getBatteryCurrent();        // mA (+ = charging, − = discharging)
M5.Power.setChargeCurrent(100);                    // mA, default 100
M5.Power.setChargeVoltage(4200);                   // mV, default 4200
```

### Standalone AXP192 reads

```cpp
// battery voltage: reg 0x78..0x79, 12-bit, LSB = 1.1 mV
uint16_t raw = (axp192_read(0x78) << 4) | (axp192_read(0x79) & 0x0F);
float vbat = raw * 1.1f / 1000.0f;                 // volts

// charging bit: reg 0x00 bit 2
bool charging = axp192_read(0x00) & 0x04;

// battery level (same formula M5Unified uses):
int pct;
if (raw > 3150) pct = (raw - 3075) * 0.16f;
else if (raw > 2690) pct = (raw - 2690) * 0.027f;
else pct = 0;
if (pct > 100) pct = 100;
```

### Battery quirks

- `getBatteryLevel()` is derived from voltage (not coulomb-counted) — **it lags by 30 s** after charge/discharge changes.
- `setChargeCurrent()` max is 390 mA, min 100 mA (StickC has a ~80 mAh cell so don't push it past 200 mA).
- Power-key hold (~6 s) → AXP192 shuts off the board. Short click → `M5.BtnPWR.wasPressed()`.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StickC autodetect | [M5GFX.cpp:738](../../M5GFX/src/M5GFX.cpp#L738) |
| StickC pinmap | [src/M5Unified.cpp:1612](src/M5Unified.cpp#L1612) |
| Mic LDO0 callback | [src/M5Unified.cpp:605](src/M5Unified.cpp#L605) |
| AXP192 driver | [src/utility/power/AXP192_Class.cpp](src/utility/power/AXP192_Class.cpp) |
