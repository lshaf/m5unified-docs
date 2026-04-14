# Display Brightness — cross-device reference

How `lcd.setBrightness(0..255)` actually works on every M5 device, the formulas under the hood, and exactly what I²C / PWM you'd write to reproduce it without M5GFX.

---

## Summary — 7 distinct mechanisms

| # | Mechanism | Resolution | Devices |
|---|-----------|------------|---------|
| 1 | Direct GPIO PWM (LEDC 9-bit) | 256 smooth steps | M5Stack, StickCPlus2, AtomS3, StickS3, Dial, DinMeter, Cardputer, CardputerADV, VAMeter, Tab5 |
| 2 | AXP192 LDO2 voltage | ~13 steps | StickC, StickCPlus |
| 3 | AXP192 LDO3 voltage | ~13 steps | Tough, Station |
| 4 | AXP192 DCDC3 (PWM-like) | ~130 steps | Core2 v1.0 |
| 5 | AXP2101 BLDO1 / DLDO1 voltage | ~28 steps | Core2 v1.1, CoreS3, CoreS3SE |
| 6 | I²C backlight chip | 256 steps | AtomS3R (chip at `0x30`) |
| 7 | PI4IOE5V6408 GPIO (on/off only) | **2 steps** | StampPLC, ArduinoNessoN1 |
| — | SSD1306 contrast reg | 256 steps | UnitC6L (OLED — self-emissive, but "brightness" == contrast) |
| — | **No backlight** (e-paper) | N/A | Paper, PaperS3, CoreInk, AirQ |

---

## 1. Direct GPIO PWM (LEDC) — the simple case

M5GFX's `Light_PWM` class ([M5GFX/src/lgfx/v1/platforms/esp32/Light_PWM.cpp](../M5GFX/src/lgfx/v1/platforms/esp32/Light_PWM.cpp)) sets up LEDC at **9-bit resolution** and writes `duty = brightness << 1` (so 0 → 0, 255 → 510).

### Standalone code (Arduino)

```cpp
constexpr int BL_PIN = 32;       // board-specific
constexpr int BL_FREQ = 44100;   // board-specific
constexpr int BL_CH = 7;         // default in M5GFX
constexpr int PWM_BITS = 9;      // LEDC 9-bit

void bl_init() {
  ledcSetup(BL_CH, BL_FREQ, PWM_BITS);
  ledcAttachPin(BL_PIN, BL_CH);
}
void bl_set(uint8_t brightness) {
  ledcWrite(BL_CH, brightness << 1);   // 0..510
}
```

For ESP32 Arduino core ≥ 3.0 (new LEDC API):
```cpp
void bl_init(uint8_t b) {
  ledcAttach(BL_PIN, BL_FREQ, PWM_BITS);
  ledcWrite(BL_PIN, b << 1);
}
```

### Per-device PWM parameters

| Device | Pin | Freq | LEDC ch |
|--------|-----|------|---------|
| M5Stack | 32 | **44.1 kHz** | 7 |
| M5StickCPlus2 | 27 | 256 Hz | 7 |
| M5AtomS3 | 16 | 256 Hz | 7 |
| M5StickS3 | 38 | 256 Hz | 7 |
| M5Dial | 9 | **44.1 kHz** | 7 |
| M5DinMeter | 9 | 256 Hz | 7 |
| M5Cardputer | 38 | 256 Hz | 7 |
| M5CardputerADV | 38 | 256 Hz | 7 |
| M5VAMeter | 38 | 512 Hz | 7 |
| M5Tab5 | 22 | **44.1 kHz** | 7 |

### Why 44.1 kHz on some boards?

44.1 kHz is above the audible range. Boards that have audio output near the backlight routing (M5Stack's DAC, Dial's piezo, Tab5's speaker) use high-freq PWM to avoid audible "singing" from the backlight driver.

### LEDC channel conflicts

M5GFX uses LEDC **channel 7** by default. If you also drive other LEDC channels (buzzers, NeoPixel RMT, etc.), keep them on channels 0-6. The timer group picked is `(pwm_channel >> 1) & 3`, so channel 7 uses timer 3.

---

## 2. AXP192 LDO2 — StickC / StickCPlus

The ST7735/ST7789 panel and its backlight share the same rail — **LDO2**. Turning brightness to 0 also kills panel logic (there's no "logic on, LED off" option on these units).

M5GFX formula ([M5GFX.cpp:316](../M5GFX/src/M5GFX.cpp#L316)):
```cpp
uint8_t reg_val = (((brightness >> 1) + 8) / 13) + 5;  // clamp 5..15
```

- brightness 0 → disable LDO2 (reg 0x12 bit 2 = 0)
- brightness 1..255 → enable LDO2 + set voltage nibble 5..15 (2.5 V..3.3 V in 0.1 V steps)

### Standalone code

```cpp
void axp192_write(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(0x34); Wire.write(reg); Wire.write(val); Wire.endTransmission();
}
uint8_t axp192_read(uint8_t reg) {
  Wire.beginTransmission(0x34); Wire.write(reg); Wire.endTransmission(false);
  Wire.requestFrom(0x34, 1);
  return Wire.read();
}
void axp192_mod(uint8_t reg, uint8_t bits_on, uint8_t mask) {
  uint8_t v = axp192_read(reg);
  v = (v & ~mask) | (bits_on & mask);
  axp192_write(reg, v);
}

void stickc_brightness(uint8_t b) {
  if (!b) { axp192_mod(0x12, 0, 0x04); return; }   // LDO2 off
  uint8_t v = (((b >> 1) + 8) / 13) + 5;            // 5..15
  axp192_mod(0x12, 0x04, 0x04);                     // LDO2 on
  axp192_mod(0x28, v << 4, 0xF0);                   // high nibble = LDO2
}
```

---

## 3. AXP192 LDO3 — Tough / Station

Same chip, same register 0x28 but **low nibble** controls LDO3. Tough's formula differs slightly:

```cpp
// M5GFX Light_M5Tough formula:
uint8_t reg_val = (b > 4) ? ((b / 24) + 5) : b;   // 5..15
```

Standalone:
```cpp
void tough_brightness(uint8_t b) {
  if (!b) { axp192_mod(0x12, 0, 0x08); return; }   // LDO3 off
  uint8_t v = (b > 4) ? ((b / 24) + 5) : b;
  axp192_mod(0x12, 0x08, 0x08);                     // LDO3 on
  axp192_mod(0x28, v, 0x0F);                        // low nibble = LDO3
}
```

Below b=5 the voltage is too low for the panel — screen goes dark.

---

## 4. AXP192 DCDC3 — M5StackCore2 v1.0

DCDC3 is a buck converter with higher-res voltage control. Formula ([M5GFX.cpp:140](../M5GFX/src/M5GFX.cpp#L140)):

```cpp
uint8_t reg_val = (brightness >> 3) + 72;   // 72..103 → ~2.5..3.3 V
```

Standalone:
```cpp
void core2_brightness(uint8_t b) {
  if (!b) { axp192_mod(0x12, 0, 0x02); return; }   // DCDC3 off
  axp192_mod(0x12, 0x02, 0x02);                     // DCDC3 on
  uint8_t v = (b >> 3) + 72;
  axp192_mod(0x27, v & 0x7F, 0x7F);                 // DCDC3 voltage
}
```

Gives ~130 visual steps (finer than LDO2/LDO3). Register 0x27's high bit is a mode flag — preserve it by masking (hence 0x7F).

---

## 5. AXP2101 BLDO1 / DLDO1 — Core2 v1.1, CoreS3(SE)

Both Core2 v1.1 and CoreS3 use AXP2101; they differ in which rail drives the backlight and its register:

| Board | Rail | Voltage reg | Enable bit |
|-------|------|-------------|------------|
| M5StackCore2 v1.1 | **BLDO1** | 0x96 | 0x90 bit 4 |
| M5StackCoreS3(SE) | **DLDO1** | 0x99 | 0x90 bit 7 |

Formula (both):
```cpp
uint8_t reg_val = (brightness + 641) >> 5;   // ~20..28 for b=1..255 → 1.9..3.3 V
```

### Standalone (CoreS3)

```cpp
void axp2101_write(uint8_t reg, uint8_t val) {
  Wire1.beginTransmission(0x34); Wire1.write(reg); Wire1.write(val); Wire1.endTransmission();
}
// ... axp2101_read / axp2101_mod same pattern ...

void cores3_brightness(uint8_t b) {
  if (b) axp2101_mod(0x90, 0x80, 0x80);          // DLDO1 enable
  else   axp2101_mod(0x90, 0,    0x80);          // DLDO1 disable
  uint8_t v = (b + 641) >> 5;
  axp2101_write(0x99, v);
}
```

### Core2 v1.1 version

Identical structure, but `reg 0x96` instead of `0x99`, and `0x10` bit instead of `0x80`:
```cpp
void core2_v11_brightness(uint8_t b) {
  if (b) axp2101_mod(0x90, 0x10, 0x10);          // BLDO1 enable
  else   axp2101_mod(0x90, 0,    0x10);          // BLDO1 disable
  uint8_t v = (b + 641) >> 5;
  axp2101_write(0x96, v);
}
```

### Detecting which revision you have

Run this once at boot — it tells you AXP192 vs AXP2101:
```cpp
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x34); Wire.write(0x03); Wire.endTransmission(false);
Wire.requestFrom(0x34, 1);
uint8_t id = Wire.read();
bool is_axp2101 = (id == 0x4A);   // AXP192 returns 0x03
```

---

## 6. I²C Backlight Chip — AtomS3R only

AtomS3R has a tiny dedicated backlight PWM driver at I²C **`0x30`** on internal bus (SDA=45, SCL=0). M5GFX source: [M5GFX.cpp:450 `Light_M5StackAtomS3R`](../M5GFX/src/M5GFX.cpp#L450).

```cpp
// one-time init
Wire1.begin(45, 0, 400000);
Wire1.beginTransmission(0x30); Wire1.write(0x00); Wire1.write(0b01000000); Wire1.endTransmission();
delay(1);
Wire1.beginTransmission(0x30); Wire1.write(0x08); Wire1.write(0b00000001); Wire1.endTransmission();
Wire1.beginTransmission(0x30); Wire1.write(0x70); Wire1.write(0b00000000); Wire1.endTransmission();

// set brightness — direct byte to reg 0x0E
void atoms3r_bl(uint8_t b) {
  Wire1.beginTransmission(0x30);
  Wire1.write(0x0E); Wire1.write(b);
  Wire1.endTransmission();
}
```

Full 8-bit resolution (256 steps), **no GPIO PWM conflicts**, and the chip handles its own soft-start.

---

## 7. PI4IOE5V6408 I/O Expander — StampPLC & ArduinoNessoN1

Only **on/off**, no dimming.

### StampPLC — expander `0x43` GPIO7, **active-LOW**

```cpp
// one-time init (set GPIO7 as output, push-pull, no pull)
pi4io_bit_on (0x43, 0x03, 1 << 7);   // direction = output
pi4io_bit_off(0x43, 0x0D, 1 << 7);   // no pull
pi4io_bit_off(0x43, 0x07, 1 << 7);   // push-pull

void stampplc_bl(uint8_t b) {
  if (b == 0) pi4io_bit_on (0x43, 0x05, 1 << 7);   // HIGH = BL off
  else        pi4io_bit_off(0x43, 0x05, 1 << 7);   // LOW  = BL on
}
```

### ArduinoNessoN1 — expander `0x44` GPIO6, **active-HIGH**

```cpp
void nesson1_bl(uint8_t b) {
  if (b) pi4io_bit_on (0x44, 0x05, 1 << 6);
  else   pi4io_bit_off(0x44, 0x05, 1 << 6);
}
```

If you need dimming on these boards: external hardware only (e.g. add a FET driven by a PWM GPIO and pipe it to the BL+ rail).

PI4IOE5V6408 register cheat-sheet:
- `0x03` direction (1 = output)
- `0x05` output value
- `0x07` push-pull (0) / open-drain (1)
- `0x0B` pull enable
- `0x0D` pull direction (0 = down, 1 = up)
- `0x0F` input value

---

## SSD1306 contrast — UnitC6L

OLED is self-emissive — there's no "backlight". The SSD1306 controller has a **contrast** register (command `0x81`) that sets pixel drive current, which visually maps to "brightness":

```
// SSD1306 command mode (via SPI):
send: 0x81, <contrast 0..0xFF>
```

M5GFX maps `setBrightness(0..255)` directly to this byte. 0 = pixels off (black), 255 = max current (brightest).

---

## E-paper devices — no backlight at all

Paper, PaperS3, CoreInk, AirQ have reflective e-paper panels. There is no backlight, and `setBrightness()` is a no-op. "Dim" modes are:

1. `setEpdMode(epd_mode_t::epd_fastest)` — fastest refresh (~400 ms), high ghosting.
2. `setEpdMode(epd_mode_t::epd_fast)` — balanced.
3. `setEpdMode(epd_mode_t::epd_quality)` — full 16-gray, slowest.
4. `setEpdMode(epd_mode_t::epd_text)` — optimized for text.

---

## Common pitfalls

### Flash-then-dim on boot

All LCDs power up with the backlight full-on until the panel is initialized. Workaround:
```cpp
lcd.setBrightness(0);    // BEFORE init()
lcd.init();
lcd.fillScreen(TFT_BLACK);
lcd.setBrightness(128);
```
M5Unified does this internally ([src/M5Unified.hpp:343](../src/M5Unified.hpp#L343)).

### Minimum usable brightness on PMIC boards

StickC/CPlus (LDO2): below brightness ~40 the rail drops below the panel's min voltage and the screen goes blank — even though logic is still powered. There's no sub-threshold dimming.

Tough (LDO3): brightness ≤ 4 → screen fully off (it's the voltage floor, not a cutoff).

### Touch-screen boards: RST line shared

Paper/PaperS3: the IT8951/EPD controller RST and GT911 touch RST share GPIO23. Toggling touch reset also resets the panel — call `setEpdMode()` after any touch reset.

### LEDC channel 7 collision

If you use `Adafruit_NeoPixel` with `ledcAttach` or drive your own PWM on channel 7, you'll clobber the backlight. Use channels 0-6 for everything else.
