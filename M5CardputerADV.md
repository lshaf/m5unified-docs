# M5CardputerADV — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5CardputerADV` (24)
**SoC:** ESP32-S3

"Advanced" Cardputer: adds **ES8311 audio codec** + **IMU (MPU6886/BMI270)** + **PCF8563 RTC**, keeps the same form factor.

---

## What's new vs [M5Cardputer](M5Cardputer.md)

| Item | Cardputer | CardputerADV |
|------|-----------|--------------|
| Audio amp | always-on | **ES8311-gated (I²C init required)** |
| IMU | none | MPU6886 or BMI270, I²C `0x68`/`0x69` |
| RTC | none | PCF8563 @ `0x51` |
| Internal I²C (for new chips) | none | **SCL=9, SDA=8** |

Display, keyboard, SD, WS2812, battery — all identical to Cardputer.

---

## Pin map (additions)

| Purpose | GPIO |
|---------|------|
| **Internal I²C SCL / SDA** | **9 / 8** (ES8311, IMU, RTC all here) |
| I²S MCK (added) | (same BCK=41, WS=43, DOUT=42 as Cardputer) |
| PDM MCK | (added — check schematic) |

Grove, SD, LCD, keyboard, WS2812 pins same as [M5Cardputer.md](M5Cardputer.md).

---

## 1. Speaker (ES8311 enable sequence)

Unlike Cardputer, **you must initialize ES8311 before sending I²S samples.** Callback source: [src/M5Unified.cpp:675 `_speaker_enabled_cb_cardputer_adv`](src/M5Unified.cpp#L675).

```cpp
#include <Wire.h>
void es8311_enable_output() {
  uint8_t seq[][2] = {
    {0x00,0x80},{0x01,0xB5},{0x02,0x18},{0x0D,0x01},
    {0x12,0x00},{0x13,0x10},{0x32,0xBF},{0x37,0x08},
  };
  for (auto& r : seq) {
    Wire1.beginTransmission(0x18);   // ES8311 I²C addr
    Wire1.write(r[0]); Wire1.write(r[1]);
    Wire1.endTransmission();
  }
}
void setup() {
  Wire1.begin(8 /* SDA */, 9 /* SCL */, 100000);
  es8311_enable_output();
  // then start I²S as in Cardputer...
}
```

---

## 2. Microphone (ES8311 ADC enable sequence)

Callback: [src/M5Unified.cpp:_microphone_enabled_cb_cardputer_adv](src/M5Unified.cpp).

```cpp
uint8_t mic_seq[][2] = {
  {0x00,0x80},{0x01,0xBA},{0x02,0x18},{0x0D,0x01},{0x0E,0x02},
  {0x14,0x10},{0x17,0xFF},{0x1C,0x6A}
};
// bulk-write the same way as above.
```

---

## 3. IMU

MPU6886 or BMI270 at `0x68` on the new internal bus (SCL=9, SDA=8). Use the corresponding driver from [src/utility/imu/](src/utility/imu/).

---

## 4. RTC

PCF8563 at `0x51` on the same internal bus.

---

## Brightness (LCD backlight)

Same as Cardputer — direct PWM on **GPIO38, LEDC ch7, 256 Hz**.

## Volume (ES8311 codec speaker)

Two layers — unlike base Cardputer.

**Software** (default): `magnification = 16`.
```cpp
M5.Speaker.setVolume(128);
```

**Hardware (ES8311 reg `0x32`)** — each step ≈ 0.5 dB, M5Unified sets `0xBF` (0 dB) at enable:
```cpp
void es8311_set_vol(uint8_t v) {
  // CardputerADV internal I²C: SDA=8, SCL=9
  Wire1.beginTransmission(0x18);
  Wire1.write(0x32); Wire1.write(v);
  Wire1.endTransmission();
}
// +6dB: es8311_set_vol(0xDF)
// mute: es8311_set_vol(0x00)
```

---

## Power management

Same as Cardputer — ADC on GPIO10, 2:1 divider. See [M5Cardputer.md § Power management](M5Cardputer.md#power-management).

---

## Source references

| Topic | File : line |
|-------|-------------|
| Speaker cb | [src/M5Unified.cpp:675](src/M5Unified.cpp#L675) |
| CardputerADV pinmap | [src/M5Unified.cpp:1675](src/M5Unified.cpp#L1675), [:2089](src/M5Unified.cpp#L2089) |
| Autodetect branch | [M5GFX.cpp:1703](../../M5GFX/src/M5GFX.cpp#L1703) |
