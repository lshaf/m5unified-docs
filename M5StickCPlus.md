# M5StickCPlus — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StickCPlus` (4)
**SoC:** ESP32-PICO-D4

Same form factor as [M5StickC](M5StickC.md); differences: **larger 135×240 ST7789 LCD** + **passive buzzer** on GPIO2.

---

## Differences from [M5StickC](M5StickC.md)

| Item | StickC | StickCPlus |
|------|--------|------------|
| LCD | ST7735 80×160 | **ST7789 135×240** |
| Buzzer | HAT only | **GPIO2 onboard** |
| Everything else | same | same |

---

## Pin map

Identical to StickC except:

| Purpose | GPIO |
|---------|------|
| LCD | ST7789, same SPI pins (MOSI=15, SCLK=13, CS=5, DC=23, RST=18) |
| LCD backlight | AXP192 LDO2 |
| **Buzzer** | **GPIO2** (PWM) |

---

## 1. Display

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() { lcd.init(); lcd.setBrightness(128); }
```

If going without M5GFX: LovyanGFX `Panel_ST7789`, same bus as StickC. Offsets: `offset_x = 52`, `offset_y = 40` (ST7789 240×240 die, showing 135×240 window).

Autodetect: [M5GFX.cpp:752](../../M5GFX/src/M5GFX.cpp#L752).

---

## 2. Buzzer

```cpp
void tone_ms(int freq, int ms) {
  ledcSetup(0, freq, 10);
  ledcAttachPin(2, 0);
  ledcWrite(0, 512);
  delay(ms);
  ledcWrite(0, 0);
  ledcDetachPin(2);
}
```

No PMIC rail needed — the buzzer is driven directly from GPIO2.

---

## 3. Mic / IMU / RTC / AXP192 / Buttons

Identical to [M5StickC.md](M5StickC.md). Backlight still needs AXP192 LDO2 enabled.

---

## Brightness (LCD backlight)

Identical mechanism to [M5StickC § Brightness](M5StickC.md#brightness-lcd-backlight) — **AXP192 LDO2**, same formula, same ~13 discrete steps. Copy the `stickc_brightness()` helper verbatim.

## Volume (buzzer)

Buzzer on GPIO2 — volume is **PWM duty cycle**, set indirectly via `magnification`. Software path:

```cpp
M5.Speaker.setVolume(128);              // master, 0..255
M5.Speaker.setAllChannelVolume(128);
```

M5Unified defaults for all buzzer boards: `magnification = 48` (buzzers need more headroom). To make louder, increase `magnification`; to make quieter, decrease master or channel volume.

Standalone buzzer tone control (bypassing M5Unified) — volume = duty:
```cpp
void tone_loud (int freq, int ms) { ledcWriteTone(0, freq); delay(ms); ledcWrite(0, 0); }
void tone_quiet(int freq, int ms) {
  ledcSetup(0, freq, 10);
  ledcAttachPin(2, 0);
  ledcWrite(0, 64);     // ~6% duty = quieter than 50%
  delay(ms);
  ledcWrite(0, 0);
}
```

The piezo's SPL peaks around 50% duty and drops symmetrically — you cannot exceed the peak by "cranking harder".

---

## Power management

Identical AXP192 path to StickC — see [M5StickC.md § Power management](M5StickC.md#power-management). Same registers, same formulas. Battery is 120 mAh on StickCPlus (vs 80 mAh on StickC) — you can run `setChargeCurrent(200)` safely.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StickCPlus autodetect | [M5GFX.cpp:752](../../M5GFX/src/M5GFX.cpp#L752) |
| StickCPlus pinmap | [src/M5Unified.cpp:1613](src/M5Unified.cpp#L1613) |
