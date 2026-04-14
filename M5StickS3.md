# M5StickS3 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StickS3` (26)
**SoC:** ESP32-S3

S3 generation of the Stick form factor. Uses a **PY32-based PMIC** (firmware on a small PY32 micro exposed over I²C) at `0x6E` and an **ES8311 codec** for speaker + mic.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD (135×240) | Yes, direct PWM BL | M5GFX |
| ES8311 codec (I²S in + out) | Yes, I²C `0x18` | Wire + register writes |
| External class-D amp | Yes, enable bit on PY32 PMIC | Wire |
| MPU6886 / BMI270 IMU | Yes, `0x68`/`0x69` | driver |
| PCF8563 RTC | Yes, `0x51` | driver |
| **PY32 PMIC** | Yes, `0x6E` | raw Wire reads |
| Battery, charge, PWRKEY | via PY32 PMIC | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / SCLK | 39 / 40 |
| LCD CS / DC / RST | 41 / 45 / 21 |
| LCD BL (**direct PWM!**) | 38 (LEDC ch7 @256 Hz) |
| I²S MCK / BCK / WS | 18 / 17 / 15 |
| I²S DATA-OUT (spk) | 14 |
| I²S DATA-IN (mic) | 16 |
| Internal I²C SCL / SDA | 48 / 47 |
| Grove (Port A) SCL / SDA | 10 / 9 |
| BtnA / BtnB | 11 / 12 |

---

## 1. PMIC init (mandatory for speaker)

The PY32 PMIC at `0x6E` holds the **amp-enable bit** and handles power-key readout. M5Unified does this init once at begin (see [src/M5Unified.cpp:1715](src/M5Unified.cpp#L1715)):

```cpp
void pm1_bit_off(uint8_t reg, uint8_t mask) { ... }
void pm1_bit_on (uint8_t reg, uint8_t mask) { ... }

void py32pmic_init() {
  Wire1.begin(47, 48, 400000);
  pm1_bit_off(0x16, 1 << 3);    // GPIO3 pull-up disabled
  pm1_bit_on (0x10, 1 << 3);    // GPIO3 = output
  pm1_bit_off(0x13, 1 << 3);    // GPIO3 = push-pull (not open-drain)
  pm1_bit_off(0x11, 1 << 3);    // GPIO3 = LOW (amp off at boot)
}
```

---

## 2. Display

Easy path — **no PMIC involved** for LCD:
```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() { lcd.init(); lcd.setBrightness(128); }
```

Autodetect: [M5GFX.cpp:1928](../../M5GFX/src/M5GFX.cpp#L1928).

Backlight is plain GPIO38 PWM — works even without PMIC writes.

---

## 3. Speaker

Enable callback: [src/M5Unified.cpp:449 `_speaker_enabled_cb_sticks3`](src/M5Unified.cpp#L449).

Two-step:
1. `pm1_bit_on(0x11, 1 << 3)` → PY32 PMIC drives amp-enable pin HIGH
2. ES8311 bulk-write to configure DAC output:
```cpp
uint8_t seq[][2] = {
  {0x00,0x80},{0x01,0xB5},{0x02,0x18},{0x0D,0x01},
  {0x12,0x00},{0x13,0x10},{0x32,0xBF},{0x37,0x08},
};
for (auto& r : seq) {
  Wire1.beginTransmission(0x18);
  Wire1.write(r[0]); Wire1.write(r[1]);
  Wire1.endTransmission();
}
```
Then I²S on MCK=18, BCK=17, WS=15, DOUT=14 on `I2S_NUM_0`.

Disable: `pm1_bit_off(0x11, 1 << 3)` (amp off; ES8311 retains config).

---

## 4. Microphone

Enable callback: [src/M5Unified.cpp:797](src/M5Unified.cpp#L797).

Different register set on same ES8311:
```
{0x00,0x80},{0x01,0xBA},{0x02,0x18},{0x0D,0x01},{0x0E,0x02},
{0x14,0x10},{0x17,0xFF},{0x1C,0x6A}
```
Uses a **temporary I²C switcher** that rebinds the I²C peripheral to the internal bus (SDA=47, SCL=48) for the duration of the write, then restores. If you stick to `Wire1` permanently on those pins, you can skip the switcher and write directly.

Then I²S RX on MCK=18, BCK=17, WS=15, DIN=16 on `I2S_NUM_1`.

---

## 5. PWRKEY / battery / IMU / RTC

- Power key read: `Power.getKeyState()` in M5Unified reads PY32 PMIC register. Standalone equivalent: read PY32 reg `0x17` (state), translate to click/hold.
- Battery voltage: PY32 PMIC exposes ADC registers.
- IMU: BMI270 or MPU6886 at `0x68`/`0x69` on internal bus.
- RTC: PCF8563 at `0x51`.

Drivers:
- [src/utility/power/PY32PMIC_Class.cpp](src/utility/power/PY32PMIC_Class.cpp) — full PY32 register map
- [src/utility/imu/](src/utility/imu/)
- [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp)

---

## Brightness (LCD backlight)

Direct GPIO PWM — no PMIC involved. M5GFX configures **LEDC channel 7, 256 Hz, 9-bit (0..511)** and writes `duty = brightness << 1`.

```cpp
// standalone (no M5GFX):
constexpr int BL_PIN = 38, BL_CH = 7;
void bl_init() { ledcSetup(BL_CH, 256, 9); ledcAttachPin(BL_PIN, BL_CH); }
void bl_set(uint8_t b) { ledcWrite(BL_CH, b << 1); }   // b = 0..255
```

With M5GFX: `lcd.setBrightness(128);`

## Volume (speaker)

Two layers — software DSP **and** hardware ES8311 DAC register. M5Unified only uses the software path by default; the ES8311's internal DAC is set to 0 dB (`reg 0x32 = 0xBF`) at enable time.

**Software (always works, per-sample):**

Effective gain = `magnification × (master/255)² × (channel/255)²`. StickS3 config sets `magnification = 1` (ES8311 provides the real gain), default `master_volume = 64`, default per-channel = 64.

```cpp
M5.Speaker.setVolume(0..255);              // master; default 64
M5.Speaker.setChannelVolume(ch, 0..255);   // per virtual channel; 8 channels
M5.Speaker.setAllChannelVolume(0..255);
```

Standalone equivalent in your DSP loop:
```cpp
int mag = 1;                        // StickS3 default
float m = master / 255.0f, c = chan / 255.0f;
int16_t scaled = sample * mag * m * m * c * c;   // clamp to int16
```

**Hardware (ES8311 `reg 0x32`, each step ≈ 0.5 dB, 0xBF ≈ 0 dB, 0xFF = +12 dB):**

Raise for extra headroom. `M5.Speaker.setVolume()` does NOT touch this — it stays at 0 dB. You write it yourself:
```cpp
void es8311_set_dac_vol(uint8_t v) {
  // note: on StickS3, internal I²C is GPIO SDA=47, SCL=48
  Wire1.beginTransmission(0x18);
  Wire1.write(0x32); Wire1.write(v);
  Wire1.endTransmission();
}
es8311_set_dac_vol(0xFF);    // +12 dB — only if software volume is saturated
```

If you hear distortion, **lower hardware reg first** (down to 0xBF), then lower software volume.

## Power management

PMIC: **PY32-based custom PMIC (a.k.a. M5PM1)** at I²C `0x6E` on internal bus. Not a standard chip — all register definitions are M5-specific. Battery voltage is read through the PM1 (which has its own ADC).

### With M5Unified

```cpp
int pct  = M5.Power.getBatteryLevel();      // 0..100, derived from voltage
int mv   = M5.Power.getBatteryVoltage();
auto chg = M5.Power.isCharging();           // from PM1 reg 0x12 bit 0
```

### Standalone — charging status (simplest)

PM1 routes the charger's STAT pin to one of its GPIOs. M5Unified reads reg `0x12` bit 0 — **LOW = charging, HIGH = not charging**:

```cpp
Wire1.begin(47, 48, 400000);                 // internal I²C pins
Wire1.beginTransmission(0x6E); Wire1.write(0x12); Wire1.endTransmission(false);
Wire1.requestFrom(0x6E, 1);
uint8_t r = Wire1.read();
bool charging = !(r & 0x01);
```

### Standalone — battery voltage

M5Unified kicks the PM1 to measure the ADC on GPIO0 of the PM1, then reads back over I²C. The exact register sequence is non-trivial — safest path for standalone is:
1. Use M5Unified's `Power` class directly (`M5.Power.begin()` + `getBatteryVoltage()`) even if you're not using the full `M5.begin()`.
2. Or copy [src/utility/power/PY32PMIC_Class.cpp](src/utility/power/PY32PMIC_Class.cpp) verbatim into your project.

### Power key / shutdown

Handled by the PM1 — hold `BtnPWR` for ~2 s to trigger a hardware power-off. Register read: [src/utility/power/PY32PMIC_Class.cpp](src/utility/power/PY32PMIC_Class.cpp) `getKeyState()`.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StickS3 autodetect | [M5GFX.cpp:1928](../../M5GFX/src/M5GFX.cpp#L1928) |
| StickS3 pinmap + PY32 init | [src/M5Unified.cpp:1711](src/M5Unified.cpp#L1711), [:1944](src/M5Unified.cpp#L1944) |
| Speaker cb | [src/M5Unified.cpp:449](src/M5Unified.cpp#L449) |
| Mic cb | [src/M5Unified.cpp:797](src/M5Unified.cpp#L797) |
| PY32 PMIC driver | [src/utility/power/PY32PMIC_Class.cpp](src/utility/power/PY32PMIC_Class.cpp) |
