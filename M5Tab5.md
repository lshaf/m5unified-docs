# M5Tab5 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5Tab5` (22)
**SoC:** ESP32-P4 (dual-core RISC-V @ 400 MHz, MIPI-DSI, MIPI-CSI, dual-channel PSRAM)

Large 5" 720×1280 tablet. The most complex board in M5Unified. Two PI4IOE5V6408 I/O expanders (`0x43`, `0x44`) gate every major rail.

---

## What's onboard

| Component | Presence | How it's gated |
|-----------|----------|----------------|
| MIPI-DSI LCD (720×1280) | Yes, 4-lane @ 960 Mbps | RST via PI4IO #2; BL = direct GPIO22 PWM |
| Cap-touch (large panel) | Yes | I²C |
| I²S stereo speaker (ES8388 + class-D amp) | Yes | codec init + PI4IO #1 GPIO1 = amp enable |
| I²S stereo mic (ES7210) | Yes | codec init |
| MIPI-CSI camera | Yes | CSI — not M5Unified |
| BMI270 + mag IMU | Yes | I²C |
| **RX8130 RTC** (not PCF8563!) | Yes, `0x32` | I²C |
| Battery + charger + USB-PD | Yes | PMIC-like subsystem |
| MicroSD | Yes | SPI |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD BL | 22 (LEDC ch7 PWM @44.1 kHz) |
| LCD RST | **PI4IO #2 (`0x44`) GPIO4** |
| I²S MCK / BCK / WS | 30 / 27 / 29 |
| I²S DATA-OUT (spk) | 26 |
| I²S DATA-IN (mic) | 28 |
| Internal I²C SCL / SDA | 32 / 31 |
| Grove (Port A) SCL / SDA | 54 / 53 |
| Port B pin1/pin2 | 17 / 52 |
| Port C pin1/pin2 | 7 / 6 |
| SD SPI SCLK/MOSI/MISO/CS | 43 / 44 / 39 / 42 |

---

## 1. Display (MIPI-DSI — not trivial standalone)

MIPI-DSI on ESP32-P4 requires specific ESP-IDF driver setup (`esp_lcd_new_panel_dsi_bus()` etc.). Strongly recommend using M5GFX which handles the lane config, panel init sequence, and backlight.

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  lcd.init();
  lcd.setBrightness(128);
}
```

Autodetect: [M5GFX.cpp:2017](../../M5GFX/src/M5GFX.cpp#L2017). Includes PI4IO #2 GPIO4 toggle to release LCD RST.

---

## 2. Speaker (ES8388 + PI4IO amp)

Callback: [src/M5Unified.cpp:486 `_speaker_enabled_cb_tab5`](src/M5Unified.cpp#L486).

Two-step:
1. Bulk-write ES8388 (`0x10`) ~25 registers (reset, DAC power-up, I²S mode, volume, mixer). Full table in the callback.
2. `PI4IO #1 (0x43)` reg `0x05` bit 1 HIGH → amp VCC on.

```cpp
void pi4io_bit_on(uint8_t dev_addr, uint8_t reg, uint8_t mask);  // same helper pattern

void speaker_on() {
  // ... (copy 25-register es8388 init from callback source) ...
  pi4io_bit_on(0x43, 0x05, 0b00000010);
}
```

Then I²S TX on MCK=30, BCK=27, WS=29, DOUT=26, `I2S_NUM_0`.

---

## 3. Microphone (ES7210 stereo ADC)

Callback: [src/M5Unified.cpp: `_microphone_enabled_cb_tab5`](src/M5Unified.cpp).

Reset ES7210 (reg `0x00 = 0xFF`), then bulk-write ~28 registers — identical structure to CoreS3's ES7210 init, same mic-bias/gain/HPF setup. Copy from source.

Then I²S RX on MCK=30, BCK=27, WS=29, DIN=28, `I2S_NUM_0` (full-duplex).

---

## 4. RX8130 RTC (NOT PCF8563!)

```cpp
#include <Wire.h>
#define RX_ADDR 0x32
Wire1.begin(31, 32, 400000);
Wire1.beginTransmission(RX_ADDR); Wire1.write(0x10); Wire1.endTransmission(false);
Wire1.requestFrom(RX_ADDR, 7);  // sec/min/hour/week/day/month/year BCD
```

Driver: [src/utility/rtc/RX8130_Class.cpp](src/utility/rtc/RX8130_Class.cpp).

---

## 5. I/O expanders

- `0x43` (PI4IO #1): AMP enable (GPIO1), other rails
- `0x44` (PI4IO #2): LCD RST (GPIO4), possibly CSI power

Register map (both chips use the same register set):
- `0x03` direction / `0x05` output / `0x07` push-pull / `0x0B` pull / `0x0F` input

---

## 6. Camera (MIPI-CSI, not M5Unified)

Use ESP-IDF's `esp_cam` API with P4-specific DSI/CSI configuration. Beyond this document.

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO22, LEDC ch7, 44.1 kHz, 9-bit**. The LCD *reset* is still behind PI4IO #2 but the backlight is plain PWM.

```cpp
void bl_init() { ledcSetup(7, 44100, 9); ledcAttachPin(22, 7); }
void bl_set(uint8_t b) { ledcWrite(7, b << 1); }   // 0..510
```

## Volume (ES8388 codec speaker)

**Software** (default): `magnification = 4`, `master_volume = 64`.

**Hardware (ES8388)** — uses **inverted** registers. Volume = attenuation:
- Reg `26` (LDACVOL): 0x00 = 0 dB (loudest), 0xC0 = -96 dB (mute)
- Reg `27` (RDACVOL): same

Each step = -0.5 dB.

```cpp
void es8388_set_attenuation(uint8_t att /* 0..0xC0 */) {
  Wire1.beginTransmission(0x10);
  Wire1.write(26); Wire1.write(att);    // LDAC
  Wire1.endTransmission();
  Wire1.beginTransmission(0x10);
  Wire1.write(27); Wire1.write(att);    // RDAC
  Wire1.endTransmission();
}
es8388_set_attenuation(0x00);   // full volume (M5Unified default)
es8388_set_attenuation(0x14);   // -10 dB
es8388_set_attenuation(0xC0);   // mute
```

Combine: software attenuates small gains (1-10 dB), hardware for large cuts.

---

## Power management

**2S Li-Po (two cells in series, 7.4 V nominal)** monitored via **INA226** on internal I²C at `0x41`. Charge-status routed through **I/O expander #1 pin 6** (LOW = charging).

### With M5Unified

```cpp
int  pct = M5.Power.getBatteryLevel();       // 0..100, computed from 2S voltage
int  mv  = M5.Power.getBatteryVoltage();     // mV (total cell-pack voltage ÷ 2)
auto chg = M5.Power.isCharging();             // reliable — reads IO-expander pin 6
```

`getBatteryLevel()` internally: `Ina226.getBusVoltage() * 500` (V→mV then ÷ 2 for per-cell) and applies the standard 3300..4150 mV curve.

### Standalone — INA226 (`0x41`)

```cpp
Wire1.begin(31, 32, 400000);
uint16_t ina226_read(uint8_t reg) {
  Wire1.beginTransmission(0x41); Wire1.write(reg); Wire1.endTransmission(false);
  Wire1.requestFrom(0x41, 2);
  return (Wire1.read() << 8) | Wire1.read();
}
float pack_voltage() { return ina226_read(0x02) * 0.00125f; }   // V (2S total)
float per_cell_mv()  { return pack_voltage() * 500.0f; }         // mV per cell
```

### Charge status pin (via PI4IO #1)

Charge status is on I/O expander `0x43` GPIO6 — not a direct MCU GPIO:

```cpp
// read expander reg 0x0F, bit 6 (inverted):
uint8_t state = pi4io_read(0x43, 0x0F);
bool charging = !(state & 0b01000000);
```

### USB-PD

Tab5 negotiates USB-PD on its USB-C port. The PD controller is autonomous — you can read negotiated voltage via INA226 (bus voltage reading on VBUS line).

---

## Source references

| Topic | File : line |
|-------|-------------|
| Tab5 autodetect + DSI bring-up | [M5GFX.cpp:2017](../../M5GFX/src/M5GFX.cpp#L2017) |
| Tab5 pinmap | [src/M5Unified.cpp:1458](src/M5Unified.cpp#L1458), [:1759](src/M5Unified.cpp#L1759), [:1916](src/M5Unified.cpp#L1916) |
| Speaker cb | [src/M5Unified.cpp:486](src/M5Unified.cpp#L486) |
| RX8130 driver | [src/utility/rtc/RX8130_Class.cpp](src/utility/rtc/RX8130_Class.cpp) |
| AXP-less power mgmt | [src/utility/power/](src/utility/power/) |
