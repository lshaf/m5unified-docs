# M5StackCoreS3SE — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StackCoreS3SE` (17)
**SoC:** ESP32-S3

"Simplified Edition" of CoreS3: **drops the camera and the ES7210 microphone** ADC. Everything else is identical.

---

## Differences from [M5StackCoreS3](M5StackCoreS3.md)

| Item | CoreS3 | CoreS3SE |
|------|--------|----------|
| GC0308 camera | ✅ | **removed** |
| ES7210 microphone ADC | ✅ | **removed** |
| I²S microphone input | ✅ | **none** |
| Pinmap / PMIC / AW9523 / speaker / display / RTC / IMU / touch | identical | identical |

For all standalone code see [M5StackCoreS3.md](M5StackCoreS3.md), and **skip the Microphone section**.

---

## Brightness / Volume

**Brightness** identical to CoreS3 — AXP2101 DLDO1. See [M5StackCoreS3.md § Brightness](M5StackCoreS3.md#brightness-lcd-backlight).

**Volume** identical to CoreS3 — AW88298 reg 0x0C. See [M5StackCoreS3.md § Volume](M5StackCoreS3.md#volume-aw88298-speaker).

CoreS3SE has no mic; everything else about audio is the same.

---

## Power management

Identical to CoreS3 — AXP2101 @ `0x34`, hardware fuel gauge. See [M5StackCoreS3.md § Power management](M5StackCoreS3.md#power-management).

---

## Source references

| Topic | File : line |
|-------|-------------|
| CoreS3SE detection | [M5GFX.cpp:1307](../../M5GFX/src/M5GFX.cpp#L1307) (shared branch, differentiates by probing ES7210 presence) |
| Pinmap | same as CoreS3 |
