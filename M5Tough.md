# M5Tough — standalone code

**Board enum:** `board_M5Tough` (8)
**SoC:** ESP32

Ruggedized IP65 Core2 variant — no physical buttons, no IMU, slightly different backlight rail.

---

## Differences from [M5Stack Core2](M5StackCore2.md)

| Item | Core2 | Tough |
|------|-------|-------|
| IMU (MPU6886) | ✅ | **removed** |
| Physical BtnA/B/C | virtual (touch) | virtual (touch) |
| LCD backlight rail | AXP192 DCDC3 | **AXP192 LDO3** |
| Vibration motor | AXP192 LDO3 | — |
| PMIC | AXP192 or AXP2101 | AXP192 only |

All pin assignments (LCD SPI, touch, I²C, speaker I²S, mic PDM, SD) are identical to Core2. See [M5StackCore2.md](M5StackCore2.md) for copy-paste code.

---

## The only real code change

When you write the AXP192 init for Tough instead of Core2, **swap DCDC3 → LDO3** for the backlight:

```cpp
// Core2 (backlight = DCDC3 reg 0x27):
axp192_write(0x27, 0xFF);

// Tough (backlight = LDO3 reg 0x28 upper nibble):
axp192_write(0x28, (axp192_read(0x28) & 0x0F) | 0xC0);  // LDO3 = 3.0 V
axp192_write(0x12, axp192_read(0x12) | (1<<3));         // enable LDO3
```

Tough's AXP192 init code: [src/utility/power/AXP192_Class.cpp](src/utility/power/AXP192_Class.cpp) — search `board_M5Tough`.

---

## Buttons (touch-virtual only)

Same zones as Core2: `y ≥ 240`, `x` ranges define BtnA/B/C.

---

## Brightness (LCD backlight)

**AXP192 LDO3** (register 0x28 low nibble). ~13 discrete steps.

```cpp
void tough_brightness(uint8_t b) {
  if (!b) { axp192_mod(0x12, 0, 0x08); return; }   // LDO3 disable
  uint8_t v = (b > 4) ? ((b / 24) + 5) : b;         // 5..15 → 2.5..3.3 V
  axp192_mod(0x12, 0x08, 0x08);                     // LDO3 enable
  axp192_mod(0x28, v, 0x0F);                        // low nibble = LDO3 voltage
}
```

Below `b=5` the rail drops below the panel's minimum and it goes dark.

## Volume (I²S speaker)

Same as Core2 — software DSP only, **magnification default = 24** (slightly higher than Core2 because Tough's amp is weaker).

```cpp
M5.Speaker.setVolume(128);
M5.Speaker.setAllChannelVolume(128);
```

See [M5StackCore2.md](M5StackCore2.md#volume-i2s-speaker) for details. To bump: `cfg.magnification = 32;` before `begin()`.

---

## Power management

**AXP192 only** (no AXP2101 variant shipped for Tough). Identical API + registers to StickC/Core2 v1.0 — copy from [M5StickC.md § Standalone AXP192 reads](M5StickC.md#standalone-axp192-reads).

```cpp
int pct = M5.Power.getBatteryLevel();
bool charging = M5.Power.isCharging() == Power_Class::is_charging;
M5.Power.setChargeCurrent(360);       // Tough has ~390 mAh cell
```

---

## Source references

| Topic | File : line |
|-------|-------------|
| Tough detection + config | [M5GFX.cpp:1071](../../M5GFX/src/M5GFX.cpp#L1071) |
| Tough pinmap | [src/M5Unified.cpp:1627](src/M5Unified.cpp#L1627), [:2165](src/M5Unified.cpp#L2165) |
