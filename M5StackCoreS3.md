# M5StackCoreS3 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StackCoreS3` (10)
**SoC:** ESP32-S3 (16 MB flash, 8 MB PSRAM)

CoreS3 is the **most complex** M5Stack device short of Tab5. Five I²C chips work together to bring the display and audio up:
- **AXP2101** PMIC (`0x34`)
- **AW9523** I/O expander (`0x58`) — LCD reset, speaker-AMP enable, touch reset, etc.
- **AW88298** class-D amp (`0x36`)
- **ES7210** stereo ADC (for mics) (`0x40`)
- **BMI270** IMU (`0x69`)

+ FT6336U touch (`0x38`), PCF8563 RTC (`0x51`), GC0308 camera (`0x21`).

---

## What's onboard

| Component | Presence | Notes |
|-----------|----------|-------|
| ILI9342C LCD (320×240) | Yes | reset via AW9523, backlight via AXP2101 |
| FT6336U cap-touch | Yes | INT/RST via AW9523 |
| I²S speaker (AW88298) | Yes | amp enable via AW9523, I²S init via AW88298 regs |
| I²S stereo mic (ES7210) | Yes | init via ES7210 regs |
| BMI270 IMU | Yes | |
| PCF8563 RTC | Yes | |
| GC0308 camera | Yes | via esp32-camera (not M5Unified) |
| SD card | Yes | |
| Battery | Yes | AXP2101 |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / MISO / SCLK | 37 / 35 / 36 |
| LCD CS | 3 |
| LCD RST | **via AW9523 pin 1** (I²C) |
| LCD BL | **via AXP2101 DLDO1** (I²C) |
| I²S BCK / WS / DATA-OUT (spk) | 34 / 33 / 13 |
| I²S MCK / DATA-IN (mic) | 0 / 14 |
| Internal I²C SCL / SDA | 11 / 12 |
| Grove (Port A) SCL / SDA | 1 / 2 |
| Port B pin1/pin2 | 8 / 9 |
| Port C pin1/pin2 | 18 / 17 |
| SD SPI SCLK/MOSI/MISO/CS | 36 / 37 / 35 / 4 |
| Touch INT/RST | via AW9523 |

---

## ⚠️ Four-step power-up sequence (critical)

Missing any step = black screen / silent speaker / hung I²C.

```
1.  AXP2101  reg 0x95 bit 3 = 1    → enable ALDO4 (LCD+SD+TF logic 3.3V)
2.  AXP2101  reg 0x93 = val         → DLDO1 voltage (backlight)
              reg 0x90 bit 4 = 1    → enable DLDO1
3.  AW9523   reg 0x04 = outputs, reg 0x05 = direction  → config I/O
              pin 1 (LCD RST) drive HIGH → release reset
4.  SPI init → drive ILI9342C init sequence
```

Full sequence: [src/utility/power/AXP2101_Class.cpp](src/utility/power/AXP2101_Class.cpp) + [M5GFX.cpp:1254](../../M5GFX/src/M5GFX.cpp#L1254) (autodetect path).

**Recommendation:** use M5GFX — it runs the entire sequence for you.

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  lcd.init();         // brings up PMIC + AW9523 + LCD
  lcd.setBrightness(128);
}
```

---

## 1. Speaker (AW88298)

Enable callback: [src/M5Unified.cpp:417 `_speaker_enabled_cb_cores3`](src/M5Unified.cpp#L417).

Two-step enable:
1. `AW9523` reg `0x02` bit 2 HIGH → raises AMP_EN line
2. `AW88298` bulk-write to config for your sample rate (~5 registers — full table in the callback, table of sample-rate → reg value in the source)

```cpp
#include <Wire.h>
void aw9523_bit_on(uint8_t reg, uint8_t mask) {
  Wire1.beginTransmission(0x58); Wire1.write(reg); Wire1.endTransmission(false);
  Wire1.requestFrom(0x58, 1); uint8_t v = Wire1.read();
  v |= mask;
  Wire1.beginTransmission(0x58); Wire1.write(reg); Wire1.write(v); Wire1.endTransmission();
}
void aw88298_write16(uint8_t reg, uint16_t val) {
  Wire1.beginTransmission(0x36);
  Wire1.write(reg); Wire1.write(val >> 8); Wire1.write(val & 0xFF);
  Wire1.endTransmission();
}
void speaker_enable() {
  aw9523_bit_on(0x02, 0b00000100);      // AMP_EN
  aw88298_write16(0x04, 0x4040);        // I²S enable, amp unmute, not power-down
  aw88298_write16(0x05, 0x0008);
  aw88298_write16(0x06, 0x14CE);        // clock config for 44.1 kHz (see reg0x06 table)
  aw88298_write16(0x0C, 0x0064);        // volume (full)
}
```

Then start I²S on BCK=34, WS=33, DOUT=13.

Disable = `aw88298_write16(0x04, 0x4000); aw9523_bit_off(0x02, 0b100);`.

---

## 2. Microphone (ES7210 stereo ADC)

Enable callback: [src/M5Unified.cpp:616 `_microphone_enabled_cb_cores3`](src/M5Unified.cpp#L616).

Reset first (`ES7210 reg 0x00 = 0xFF`), then bulk-write ~28 registers configuring 4-channel mic bias at 2.5 V, gain 0x1B, HPF, PDN (mic1+2 active, mic3+4 off).

**Copy the full `data[]` array from the callback source**; the sequence is too long to inline here and does not summarize usefully.

Then start I²S RX on MCK=0, BCK=34, WS=33, DIN=14 (full-duplex with speaker on `I2S_NUM_1`).

---

## 3. BMI270 IMU (`0x69`)

Axis fix-up rule applied by M5Unified (when BMI270 is at `0x69`): invert **mag Y and Z**. Accel/gyro used as-is.

Use Bosch `BMI270_SensorAPI` — it needs the ~8 KB config blob upload via reg `0x5E` before the sensor starts.

---

## 4. Touch (FT6336U `0x38`)

On internal I²C. INT and RST routed through AW9523 — M5Unified's Touch class handles the AW9523 toggles. Standalone: configure AW9523 once so the reset line is HIGH, then talk I²C directly to FT6336U.

---

## 5. Buttons (virtual)

Same as Core2: `touch.y ≥ 240`, x-range → BtnA/B/C. No physical buttons.

---

## Brightness (LCD backlight)

**AXP2101 DLDO1** — register 0x99 voltage, enable bit on register 0x90 bit 7.

```cpp
void cores3_brightness(uint8_t b) {
  if (b) axp2101_mod(0x90, 0x80, 0x80);     // DLDO1 enable
  else   axp2101_mod(0x90, 0,    0x80);     // DLDO1 disable
  uint8_t v = (b + 641) >> 5;                // ~20..28 = 1.9..3.3 V
  axp2101_write(0x99, v);
}
```

5-bit voltage control → ~28 discrete steps. With M5GFX: `lcd.setBrightness(0..255)`.

## Volume (AW88298 speaker)

Two layers:

**Software (M5Unified default):** `magnification = 4`, `master_volume = 64`. AW88298 register is left at the M5Unified-configured `0x0C = 0x0064` (≈ mid gain).

```cpp
M5.Speaker.setVolume(128);
M5.Speaker.setAllChannelVolume(128);
```

**Hardware (AW88298 register `0x0C` — 16-bit write):**

The AW88298 is a 16-bit-register amp. Bit-field in `0x0C`: high byte = mute bit, low byte = volume step.

```cpp
void aw88298_set_volume(uint8_t vol /* 0..0xFF */) {
  uint8_t hi = 0;          // unmuted
  uint8_t lo = vol;
  Wire1.beginTransmission(0x36);
  Wire1.write(0x0C);
  Wire1.write(hi); Wire1.write(lo);
  Wire1.endTransmission();
}
aw88298_set_volume(0xFF);   // max hardware gain
aw88298_set_volume(0x00);   // mute at hardware level
```

**Order of attenuation matters:** prefer software volume down + hardware max, rather than hardware low + software high — software clipping is cleaner than hardware distortion.

---

## Power management

PMIC: **AXP2101** @ I²C `0x34` on internal bus — with hardware fuel gauge (no guessing from voltage).

### With M5Unified

```cpp
int  pct      = M5.Power.getBatteryLevel();       // 0..100 % — direct from gauge
int  mv       = M5.Power.getBatteryVoltage();     // mV
auto chg      = M5.Power.isCharging();            // reliable
int  vbus     = M5.Power.getVBUSVoltage();        // USB-C mV
bool on_usb   = vbus > 4500;
M5.Power.setChargeCurrent(500);                    // mA (CoreS3 has 500 mAh cell)
M5.Power.setChargeVoltage(4200);                   // mV
```

### Standalone AXP2101

```cpp
// battery % — register 0xA4 direct read:
uint8_t pct = axp2101_read(0xA4);                  // 0..100

// battery voltage — 14-bit (reg 0x34 low 6 bits | reg 0x35):
uint16_t raw = ((axp2101_read(0x34) & 0x3F) << 8) | axp2101_read(0x35);
float vbat = raw / 1000.0f;                        // volts

// charging status — reg 0x01 bits 5-6:
//   01 = charging, 10 = discharging, 00 = standby
uint8_t s = axp2101_read(0x01);
bool charging    = (s & 0b01100000) == 0b00100000;
bool discharging = (s & 0b01100000) == 0b01000000;

// VBUS good:
bool vbus_present = axp2101_read(0x00) & 0x20;

// power key (reg 0x49 bits 2-3, write to clear):
uint8_t pek = (axp2101_read(0x49) >> 2) & 0x03;
if (pek) axp2101_write(0x49, pek << 2);
```

### USB-C PD quirks

CoreS3 can sink up to 5 V / 3 A via USB-C. When VBUS is present, the battery is **always** charged (AXP2101 manages the handoff). To force-disable charging (e.g. to calibrate level while running on USB):
```cpp
M5.Power.setBatteryCharge(false);
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| CoreS3 autodetect + PMIC bring-up | [M5GFX.cpp:1254](../../M5GFX/src/M5GFX.cpp#L1254) |
| CoreS3 pinmap | [src/M5Unified.cpp:1775](src/M5Unified.cpp#L1775), [:1931](src/M5Unified.cpp#L1931) |
| Speaker callback | [src/M5Unified.cpp:417](src/M5Unified.cpp#L417) |
| Mic callback | [src/M5Unified.cpp:616](src/M5Unified.cpp#L616) |
| AXP2101 driver | [src/utility/power/AXP2101_Class.cpp](src/utility/power/AXP2101_Class.cpp) |
