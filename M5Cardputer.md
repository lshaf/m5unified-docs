# M5Cardputer — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5Cardputer` (14)
**SoC:** ESP32-S3 (8 MB PSRAM)

Credit-card-sized computer: **56-key QWERTY keyboard** + 1.14" ST7789 LCD + I²S speaker + PDM mic + SD + battery.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD (135×240) | Yes | M5GFX OR LovyanGFX |
| 56-key diode matrix keyboard | Yes | **`m5stack/M5Cardputer` library** (keyboard class) |
| I²S speaker (AW8737 amp) | Yes | ESP-IDF `i2s_std` |
| PDM mic | Yes | ESP-IDF `i2s_pdm_rx` |
| MicroSD (SPI) | Yes | `SD.h` |
| WS2812 | Yes, GPIO21 | Adafruit_NeoPixel |
| Battery + charger | Yes | ADC |
| IMU / RTC | **No** (base Cardputer) | — |

---

## Pin map (full)

| Purpose | GPIO |
|---------|------|
| LCD MOSI / SCLK | 35 / 36 |
| LCD CS / DC / RST | 37 / 34 / 33 |
| LCD BL | 38 (LEDC ch7 PWM @256 Hz) |
| Grove (Port A) SCL / SDA | 1 / 2 |
| SD SPI SCLK / MOSI / MISO / CS | 40 / 14 / 39 / 12 |
| WS2812 | 21 |
| I²S BCK / WS / DATA-OUT | 41 / 43 / 42 |
| PDM DATA-IN | 46 |
| PDM CLK (shared with I²S WS) | 43 |
| BtnA (G0 bottom) | 0 |
| Keyboard matrix | handled by M5Cardputer library |

---

## 1. Display

M5GFX autodetect: [M5GFX.cpp:1647](../../M5GFX/src/M5GFX.cpp#L1647) (shared branch that distinguishes Cardputer vs CardputerADV vs VAMeter by I²C probe).

Standalone LovyanGFX `Panel_ST7789`:
- Bus: MOSI=35, SCLK=36, DC=34, host=SPI3
- Panel: CS=37, RST=33, width=135, height=240, offset_x=52, offset_y=40, invert=true
- Light: PWM pin 38, ch7, 256 Hz

---

## 2. Keyboard

The 56-key matrix is **diode-matrix scanned** — not part of M5Unified. Use the separate library:

```cpp
#include <M5Cardputer.h>
void setup() {
  auto cfg = M5.config();
  M5Cardputer.begin(cfg);          // wraps M5Unified + keyboard init
}
void loop() {
  M5Cardputer.update();
  if (M5Cardputer.Keyboard.isChange() && M5Cardputer.Keyboard.isPressed()) {
    auto status = M5Cardputer.Keyboard.keysState();
    for (auto k : status.word) Serial.print(k);
  }
}
```

If you want the raw matrix (no M5Cardputer library): clone the [M5Cardputer repo](https://github.com/m5stack/M5Cardputer) and copy `M5Cardputer/src/utility/Keyboard.cpp`. The scanning uses column-drive output pins and row-read input pins — ~8 GPIOs total.

---

## 3. Speaker (I²S)

```cpp
#include <ESP_I2S.h>
I2SClass i2s;
void setup() {
  i2s.setPins(41 /* BCK */, 43 /* WS */, 42 /* DOUT */);
  i2s.begin(I2S_MODE_STD, 44100, I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO);
}
```

No enable callback — amp is always powered.

---

## 4. PDM microphone

```cpp
// PDM CLK shares pin with speaker WS (43). Use full-duplex on same I2S port.
// gpio_cfg.clk = GPIO_NUM_43; gpio_cfg.din = GPIO_NUM_46;
```

---

## 5. SD

```cpp
#include <SD.h>
#include <SPI.h>
SPIClass sd_spi(HSPI);
void setup() {
  sd_spi.begin(40, 39, 14, 12);
  SD.begin(12, sd_spi, 20000000);
}
```

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO38, LEDC ch7, 256 Hz, 9-bit**.

```cpp
void bl_init() { ledcSetup(7, 256, 9); ledcAttachPin(38, 7); }
void bl_set(uint8_t b) { ledcWrite(7, b << 1); }
```

## Volume (I²S speaker, no codec)

Software DSP only — the AW8737-style amp has **no I²C**. `magnification = 16` by default.

```cpp
M5.Speaker.setVolume(128);
M5.Speaker.setAllChannelVolume(128);
```

To go louder: increase `cfg.magnification` before `begin()` (e.g. 32). Too high → clips.

No hardware volume — this is a big difference from CardputerADV (which adds ES8311).

---

## Power management

ADC on **GPIO10** with 2:1 divider. Same scheme as DinMeter / AirQ.

```cpp
int pct = M5.Power.getBatteryLevel();
int mv  = M5.Power.getBatteryVoltage();
auto chg = M5.Power.isCharging();           // charge_unknown
```

Standalone:
```cpp
float vbat_mv() { return (analogRead(10) * 3300.0f / 4095.0f) * 2.0f; }
int pct_from_mv(float mv) {
  int p = (mv - 3300) * 100 / (4150 - 3350);
  return (p < 0) ? 0 : (p > 100) ? 100 : p;
}
```

Cardputer has no power-hold GPIO (it runs off USB-powered charger → cell). No software shutdown — you just let USB drop.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Cardputer autodetect branch | [M5GFX.cpp:1647](../../M5GFX/src/M5GFX.cpp#L1647) |
| Cardputer pinmap | [src/M5Unified.cpp:1674](src/M5Unified.cpp#L1674) |
| Keyboard driver (external) | [m5stack/M5Cardputer](https://github.com/m5stack/M5Cardputer) |
