# M5StackCore2 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StackCore2` (2)
**SoC:** ESP32-D0WDQ6 (16 MB flash, 8 MB PSRAM)
**Two revisions:** v1.0 uses **AXP192**, v1.1 uses **AXP2101** — auto-detected.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ILI9342C LCD (320×240) | Yes | M5GFX |
| FT6336U cap-touch | Yes, I²C `0x38` | LGFX `Touch_FT5x06` |
| I²S speaker (external amp, PMIC-gated) | Yes | ESP-IDF `i2s_std` + PMIC write |
| PDM mic | Yes | ESP-IDF `i2s_pdm_rx` |
| MPU6886 IMU | Yes, I²C `0x68` | MPU6886 driver |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| AXP192 OR AXP2101 PMIC | Yes, I²C `0x34` | [src/utility/power/](src/utility/power/) |
| SD card SPI | Yes | `SD.h` |
| Battery | Yes | PMIC |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / MISO / SCLK | 23 / 38 / 18 |
| LCD CS / DC | 5 / 15 |
| LCD RST | **no GPIO** — PMIC-controlled (AXP192 GPIO4 or AXP2101 ALDO2) |
| LCD BL | **no GPIO** — PMIC-controlled (AXP192 DCDC3 or AXP2101 BLDO1) |
| Internal I²C SCL / SDA | 22 / 21 |
| Grove (Port A) SCL / SDA | 33 / 32 |
| Port B pin1/pin2 | 36 / 26 |
| Port C pin1/pin2 | 13 / 14 |
| SD SPI SCLK/MOSI/MISO/CS | 18 / 23 / 38 / 4 |
| I²S BCK / WS / DATA-OUT (spk) | 12 / 0 / 2 |
| PDM DATA-IN / CLK (mic) | 34 / 0 |
| Touch INT | 39 |
| M5GO bottom WS2812 | 25 |

---

## ⚠️ Display NEEDS PMIC to power up

The ILI9342C has no direct GPIO for reset or backlight. You MUST write the PMIC first or the screen stays black. This is the #1 gotcha people hit when going "standalone".

### Detect which PMIC you have

```cpp
#include <Wire.h>
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x34); Wire.write(0x03); Wire.endTransmission();
Wire.requestFrom(0x34, 1);
uint8_t chip_id = Wire.read();
// AXP192 ID at reg 0x03 = 0x03; AXP2101 has a different fingerprint
// Simpler: try AXP192's reg 0x00 (POWER_STATUS) — both chips have it, values differ
```

### AXP192 path (v1.0) — minimal init

```cpp
void axp192_write(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(0x34); Wire.write(reg); Wire.write(val); Wire.endTransmission();
}
void axp192_init() {
  axp192_write(0x26, 0x6A);   // DCDC1 = 3350 mV (ESP32 VDD)
  axp192_write(0x28, 0xCC);   // LDO2 = 3.3 V (LCD+SD), LDO3 = 3.0 V (vibration)
  axp192_write(0x12, 0x04 | 0x02);  // enable LDO2 + DCDC3 (backlight)
  axp192_write(0x27, 0xFF);   // DCDC3 = max (backlight on)
  axp192_write(0x96, 0x02);   // GPIO4 HIGH = release LCD reset
}
```

After this, you can init the ILI9342C over SPI.

### AXP2101 path (v1.1)

```cpp
void axp2101_init() {
  axp2101_write(0x95, 28);     // ALDO4 = 3.3 V (LCD+SD logic)
  axp2101_write(0x90, (1<<3)); // enable ALDO4
  axp2101_write(0x96, (641>>5)+641-32); // BLDO1 ≈ 3.3 V (backlight)
  axp2101_write(0x90, (1<<3) | (1<<4)); // enable BLDO1 too
  // toggle ALDO2 to release LCD reset
  axp2101_write(0x90, (1<<3) | (1<<4));             // ALDO2 off
  delay(10);
  axp2101_write(0x90, (1<<3) | (1<<4) | (1<<1));    // ALDO2 on
}
```

Full register map + exact values: [src/utility/power/AXP192_Class.cpp](src/utility/power/AXP192_Class.cpp) and [AXP2101_Class.cpp](src/utility/power/AXP2101_Class.cpp).

---

## 1. Display (after PMIC init)

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  // M5GFX.init() calls its own PMIC-detect + power-up path internally.
  // So if you ONLY want display (no audio), this is enough:
  lcd.init();
  lcd.setBrightness(128);
}
```

If you skip M5GFX and wire your own ILI9342C driver, remember: run the AXP init **first**, then SPI.

---

## 2. Speaker (I²S + PMIC amp enable)

The amp lives after the PMIC. The "enable" callback flips PMIC GPIO2 (AXP192) or ALDO3 (AXP2101).

Standalone enable:
```cpp
// AXP192:
axp192_write(0x92, 0x00);    // GPIO2 = NMOS output mode
axp192_write(0x94, 0x04);    // GPIO2 drive HIGH → amp VCC on
// AXP2101:
axp2101_write(0x93, (3300-500)/100);     // ALDO3 = 3.3 V
axp2101_write(0x90, prev_value | (1<<2)); // enable ALDO3
```

Then I²S on BCK=12, WS=0, DOUT=2 (magnification 16 for normalized volume).

Callback source: [src/M5Unified.cpp:391](src/M5Unified.cpp#L391).

---

## 3. Microphone (PDM, always powered)

PDM DATA on GPIO34, CLK on GPIO0. No enable callback — mic is wired to the always-on 3.3 V rail.

```c
// ESP-IDF:
gpio_cfg.clk = GPIO_NUM_0;
gpio_cfg.din = GPIO_NUM_34;
// then i2s_pdm_rx mode
```

---

## 4. Touch (FT6336U @ `0x38`)

```cpp
Wire.begin(21, 22, 400000);
pinMode(39, INPUT_PULLUP);   // touch INT
// 6-byte touch packet at reg 0x02:
Wire.beginTransmission(0x38); Wire.write(0x02); Wire.endTransmission(false);
Wire.requestFrom(0x38, 7);
uint8_t n = Wire.read();       // number of touch points
uint8_t x_h = Wire.read(), x_l = Wire.read();
uint8_t y_h = Wire.read(), y_l = Wire.read();
```

Full driver: [src/utility/Touch_Class.cpp](src/utility/Touch_Class.cpp).

### Virtual A/B/C buttons

Core2's BtnA/B/C are **touch zones**, not physical buttons:
- y ≥ 240, x 0…106 → BtnA
- y ≥ 240, x 107…213 → BtnB
- y ≥ 240, x 214…320 → BtnC

---

## 5. IMU / RTC

MPU6886 at `0x68`, PCF8563 at `0x51`, both on internal I²C (21/22).

---

## Brightness (LCD backlight)

**PMIC-gated — there is no backlight GPIO.** The formula differs by revision.

### v1.0 (AXP192) — register 0x27 (DC3 voltage)

```cpp
void axp192_set_brightness(uint8_t b) {
  if (b) axp192_mod(0x12, 0x02, 0x02);      // DC3 enable
  else   axp192_mod(0x12, 0,    0x02);      // DC3 disable
  uint8_t v = (b >> 3) + 72;                 // 72..103 → ~2.5..3.3 V
  axp192_write(0x27, v & 0x7F);
}
```

### v1.1 (AXP2101) — register 0x96 (BLDO1 voltage)

```cpp
void axp2101_set_brightness(uint8_t b) {
  if (b) axp2101_mod(0x90, 0x10, 0x10);     // BLDO1 enable
  else   axp2101_mod(0x90, 0,    0x10);     // BLDO1 disable
  uint8_t v = (b + 641) >> 5;                // ~20..28
  axp2101_write(0x96, v);
}
```

With M5GFX these hide behind `lcd.setBrightness(0..255)` — it auto-detects the PMIC and picks the right path.

## Volume (I²S speaker)

Software DSP only — **no hardware volume register** on the amp. Magnification default **16**.

```cpp
M5.Speaker.setVolume(128);             // master 0..255, default 64
M5.Speaker.setAllChannelVolume(128);
```

Effective gain = `16 × (master/255)² × (chan/255)²`. If too quiet at max:
```cpp
auto cfg = M5.Speaker.config();
cfg.magnification = 32;                // was 16
M5.Speaker.config(cfg);
M5.Speaker.begin();
```

The amp itself has on/off only via PMIC (AXP192 GPIO2 or AXP2101 ALDO3) — no gain.

---

## Power management

**Two hardware variants.** `M5.Power.getType()` returns either `pmic_axp192` (v1.0) or `pmic_axp2101` (v1.1); the API is identical.

### With M5Unified

```cpp
int level = M5.Power.getBatteryLevel();       // 0..100 %
int mv    = M5.Power.getBatteryVoltage();     // mV
auto chg  = M5.Power.isCharging();
int vbus  = M5.Power.getVBUSVoltage();        // USB mV
int cur   = M5.Power.getBatteryCurrent();
M5.Power.setChargeCurrent(360);               // mA (Core2 has 390 mAh cell)
```

### v1.0 (AXP192)

Same register set as StickC. Voltage: reg `0x78..0x79` (12-bit, ×1.1 mV). Charging: reg `0x00` bit 2. See [M5StickC.md § Standalone AXP192 reads](M5StickC.md#standalone-axp192-reads) — copy verbatim.

Battery level formula (piecewise linear from voltage).

### v1.1 (AXP2101) — **hardware fuel gauge**, much better

AXP2101 has a built-in coulomb-counting gauge — you read percentage directly, no guessing:

```cpp
// battery level (reg 0xA4, directly 0..100):
uint8_t pct = axp2101_read(0xA4);

// battery voltage (reg 0x34..0x35, 14-bit, ×1 mV):
uint16_t raw = ((axp2101_read(0x34) & 0x3F) << 8) | axp2101_read(0x35);
float vbat = raw / 1000.0f;                       // volts

// charging state (reg 0x01 bits 5-6):
uint8_t s = axp2101_read(0x01);
bool charging = (s & 0b01100000) == 0b00100000;   // 01 = charging, 10 = discharging

// VBUS good flag (reg 0x00 bit 5):
bool vbus_present = axp2101_read(0x00) & 0x20;

// power key press (reg 0x49 bits 2-3; write-1-to-clear):
uint8_t pek = (axp2101_read(0x49) >> 2) & 0x03;   // 1=short, 2=long, 3=both
if (pek) axp2101_write(0x49, pek << 2);
```

### Detecting which variant

```cpp
// M5Unified does this in Power::begin() — adapt for standalone:
// Read a register that differs: AXP192 reg 0x03 returns 0x03; AXP2101 returns 0x4A.
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x34); Wire.write(0x03); Wire.endTransmission(false);
Wire.requestFrom(0x34, 1);
uint8_t id = Wire.read();
bool is_axp2101 = (id == 0x4A);
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| Core2 autodetect (distinguishes Tough/Station) | [M5GFX.cpp:885](../../M5GFX/src/M5GFX.cpp#L885) |
| Core2 pinmap | [src/M5Unified.cpp:1626](src/M5Unified.cpp#L1626) |
| AXP192 full init | [src/utility/power/AXP192_Class.cpp](src/utility/power/AXP192_Class.cpp) |
| AXP2101 full init | [src/utility/power/AXP2101_Class.cpp](src/utility/power/AXP2101_Class.cpp) |
| Speaker enable callback | [src/M5Unified.cpp:391](src/M5Unified.cpp#L391) |
