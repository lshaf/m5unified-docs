# M5Stack (Basic / Gray / Fire / GO) — standalone code

**Board enum:** `board_M5Stack` (1)
**SoC:** ESP32 (D0WDQ6)

Classic 5 cm cube Core. Four hardware variants share the same pinmap:
- **Basic** — base Core, no IMU
- **Gray** — adds MPU6886 IMU (or early SH200Q)
- **Fire** — Gray + MPU9250 / AK8963 mag + 16 MB PSRAM
- **GO** — Basic + M5GO bottom (battery, WS2812×10, IR, mic)

---

## What's onboard (base Core)

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ILI9342C LCD (320×240) | Yes | M5GFX OR LovyanGFX `Panel_ILI9342` |
| DAC speaker (GPIO25) | Yes | Arduino `dacWrite` OR `I2S_DAC` |
| BtnA / B / C (face) | Yes, GPIO39 / 38 / 37 | digitalRead |
| IP5306 charger/boost | Yes, I²C `0x75` | register writes |
| SD card SPI | Yes | `SD.h` |
| Internal + Grove I²C | Yes (shared pins!) | `Wire` |
| M5GO bottom ADC mic | GO-only, GPIO34 | `analogRead` or I²S ADC |
| MPU6886 / SH200Q IMU | Gray / Fire only | [src/utility/imu/](src/utility/imu/) |
| AK8963 mag | Fire only | raw I²C |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| LCD MOSI / MISO / SCLK | 23 / 19 / 18 |
| LCD CS / DC / RST / BL | 14 / 27 / 33 / 32 (LEDC ch7 PWM) |
| BtnA / BtnB / BtnC | 39 / 38 / 37 |
| Internal I²C SCL / SDA | 22 / 21 (**shared with Grove Port A**) |
| Grove (Port A) SCL / SDA | 22 / 21 |
| Port B pin1 / pin2 | 36 / 26 |
| Port C pin1 / pin2 | 16 / 17 |
| SD SPI SCLK / MOSI / MISO / CS | 18 / 23 / 19 / 4 |
| DAC speaker | 25 |
| M5GO WS2812 (×10) | 15 |
| IR TX | 17 (on Port C) |

### Boot quirk: GPIO15 default LOW

M5Unified forces `GPIO15 = LOW` at boot ([src/M5Unified.cpp:1562](src/M5Unified.cpp#L1562)) to suppress an LED flash that interferes with WiFi power-up. If you skip M5Unified and see flaky WiFi on cold boot, drive GPIO15 low for ~10 ms before `WiFi.begin()`.

---

## 1. Display

### M5GFX autodetect
```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  lcd.init();
  lcd.setBrightness(128);
}
```
Autodetect: [M5GFX.cpp:1103](../../M5GFX/src/M5GFX.cpp#L1103).

### LovyanGFX explicit
```cpp
#include <LovyanGFX.hpp>
class LGFX_M5Stack : public lgfx::LGFX_Device {
  lgfx::Panel_ILI9342  _panel;
  lgfx::Bus_SPI        _bus;
  lgfx::Light_PWM      _light;
public:
  LGFX_M5Stack() {
    { auto c = _bus.config();
      c.spi_host = VSPI_HOST; c.spi_mode = 0;
      c.freq_write = 40000000; c.freq_read = 16000000;
      c.spi_3wire = false; c.use_lock = true;
      c.dma_channel = SPI_DMA_CH_AUTO;
      c.pin_sclk = 18; c.pin_mosi = 23; c.pin_miso = 19; c.pin_dc = 27;
      _bus.config(c); _panel.setBus(&_bus); }
    { auto c = _panel.config();
      c.pin_cs = 14; c.pin_rst = 33;
      c.panel_width = 320; c.panel_height = 240;
      c.offset_x = 0; c.offset_y = 0;
      c.readable = true; c.invert = false;
      _panel.config(c); }
    { auto c = _light.config();
      c.pin_bl = 32; c.pwm_channel = 7; c.freq = 44100;
      _light.config(c); _panel.setLight(&_light); }
    setPanel(&_panel);
  }
};
```

---

## 2. DAC speaker (GPIO25)

```cpp
// simplest - 8-bit PWM via DAC:
for (int i = 0; i < 16000; ++i) {
  dacWrite(25, 128 + 60 * sin(2 * PI * i / 40));   // 400 Hz sine @ 16 kHz
  delayMicroseconds(62);
}
```

Or via I²S DAC mode (hardware pacing):
```cpp
#include <driver/i2s.h>
i2s_config_t cfg = {
  .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX | I2S_MODE_DAC_BUILT_IN),
  .sample_rate = 44100,
  .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
  .channel_format = I2S_CHANNEL_FMT_ONLY_RIGHT,  // right chan = GPIO25
  .communication_format = I2S_COMM_FORMAT_I2S_MSB,
  .dma_buf_count = 4, .dma_buf_len = 256,
};
i2s_driver_install(I2S_NUM_0, &cfg, 0, NULL);
i2s_set_dac_mode(I2S_DAC_CHANNEL_RIGHT_EN);
```

Magnification ≈ 8 is what M5Unified applies to normalize volume.

---

## 3. Microphone (M5GO bottom only, GPIO34)

Analog mic via ADC. Use `I2S_MODE_ADC_BUILT_IN`:
```cpp
i2s_config_t cfg = {
  .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX | I2S_MODE_ADC_BUILT_IN),
  .sample_rate = 16000,
  .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
  .channel_format = I2S_CHANNEL_FMT_ONLY_RIGHT,
  .dma_buf_count = 4, .dma_buf_len = 512,
};
i2s_driver_install(I2S_NUM_0, &cfg, 0, NULL);
i2s_set_adc_mode(ADC_UNIT_1, ADC1_CHANNEL_6);   // GPIO34 = ADC1_CH6
i2s_adc_enable(I2S_NUM_0);
```

---

## 4. IP5306 charger (I²C `0x75`)

IP5306 is a fixed-function charger/boost IC. Limited interface — mostly one register for enable:
```cpp
#include <Wire.h>
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x75);
Wire.write(0x00);             // reg 0: system config
Wire.write(0x37);             // enable boost, 5V out, auto on
Wire.endTransmission();
```

Battery is present only with M5GO bottom or M5Stack battery base attached.

---

## 5. IMU (Gray / Fire)

MPU6886 at `0x68` on Wire (shared Port A + internal):
```cpp
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x68); Wire.write(0x6B); Wire.write(0x00); Wire.endTransmission();  // wake
```
Full driver: [src/utility/imu/MPU6886_Class.cpp](src/utility/imu/MPU6886_Class.cpp).

For SH200Q fallback: [src/utility/imu/SH200Q_Class.cpp](src/utility/imu/SH200Q_Class.cpp).

---

## 6. SD card

```cpp
#include <SD.h>
SD.begin(4);   // CS = 4, uses default VSPI (18/23/19)
```

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO32, LEDC channel 7, 44.1 kHz, 9-bit**. Duty = `brightness << 1`.

```cpp
constexpr int BL_PIN = 32, BL_CH = 7;
void bl_init() { ledcSetup(BL_CH, 44100, 9); ledcAttachPin(BL_PIN, BL_CH); }
void bl_set(uint8_t b) { ledcWrite(BL_CH, b << 1); }
```

With M5GFX: `lcd.setBrightness(0..255)`.

The unusual 44.1 kHz frequency avoids audible whine that would otherwise couple into the DAC speaker on GPIO25.

## Volume (DAC speaker)

Software-only. The GPIO25 DAC is 8-bit; the whole signal chain is DSP-scaled before the DAC.

Effective gain per sample = `magnification × (master/255)² × (chan/255)²`.

M5Unified defaults for M5Stack:
- `magnification = 8` (compensates for 8-bit DAC range)
- `master_volume = 64`
- `channel_volume = 64`

```cpp
M5.Speaker.setVolume(128);               // master, 0..255
M5.Speaker.setAllChannelVolume(128);     // all virtual channels
// Or bump config for more headroom:
auto cfg = M5.Speaker.config();
cfg.magnification = 16;
M5.Speaker.config(cfg);
M5.Speaker.begin();
```

Peaking past ~200 on master + channel will clip — DAC saturates. No hardware volume register (the DAC has no attenuator).

---

## Power management

**Only with M5GO bottom or battery base.** The M5Stack Core itself has no battery. The battery bases use an **IP5306** charger/boost IC at I²C `0x75`.

### With M5Unified

```cpp
int pct = M5.Power.getBatteryLevel();     // 5 discrete levels only: 0, 25, 50, 75, 100
auto chg = M5.Power.isCharging();
M5.Power.setBatteryCharge(true);
```

### Standalone IP5306

IP5306 exposes very little — **only 5 battery levels** (2-bit encoded, not a percentage):

```cpp
Wire.begin(21, 22, 400000);
// battery level register (0x78), top nibble:
Wire.beginTransmission(0x75); Wire.write(0x78); Wire.endTransmission(false);
Wire.requestFrom(0x75, 1);
uint8_t r = Wire.read();
int pct;
switch (r >> 4) {
  case 0x00: pct = 100; break;
  case 0x08: pct = 75;  break;
  case 0x0C: pct = 50;  break;
  case 0x0E: pct = 25;  break;
  default:   pct = 0;   break;
}

// charging: register 0x70 bit 3
Wire.beginTransmission(0x75); Wire.write(0x70); Wire.endTransmission(false);
Wire.requestFrom(0x75, 1);
bool charging = Wire.read() & 0x08;
```

### Quirks

- Charge current / voltage are preset by the battery base; `setChargeCurrent()` on non-I²C IP5306 units is a **no-op** (some bases ship with the IP5306 pin-strapped for 2 A fixed).
- No VBUS voltage measurement.
- 30-second auto-off if drawing < 100 mA — disable with:
  ```cpp
  M5.Power.Ip5306.setPowerBoostKeepOn(true);
  ```
  Or standalone: set SYS_CTL0 (reg 0x00) bit 5 = 1.

---

## Source references

| Topic | File : line |
|-------|-------------|
| M5Stack autodetect | [M5GFX.cpp:1103](../../M5GFX/src/M5GFX.cpp#L1103) |
| M5Stack pinmap | [src/M5Unified.cpp:1562](src/M5Unified.cpp#L1562), [:1608](src/M5Unified.cpp#L1608) |
| DAC speaker code | [src/utility/Speaker_Class.cpp](src/utility/Speaker_Class.cpp) — search `use_dac` |
| IP5306 driver | [src/utility/power/IP5306_Class.cpp](src/utility/power/IP5306_Class.cpp) |
