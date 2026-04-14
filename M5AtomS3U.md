# M5AtomS3U — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomS3U` (138)
**SoC:** ESP32-S3

USB-A plug form factor with PDM microphone.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| PDM microphone | Yes | ESP-IDF `i2s_pdm_rx` |
| WS2812 LED (GPIO35) | Yes | Adafruit_NeoPixel |
| BtnA | Yes, GPIO41 | Arduino digitalRead |
| Grove Port A | Yes | Arduino Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| PDM DATA-IN | 38 |
| PDM CLK (`pin_ws`) | 39 |
| WS2812 | 35 |
| BtnA | 41 |
| Grove SCL / SDA | 1 / 2 |
| Internal I²C SCL / SDA | 39 / 38 (**collides with mic** — see warning) |

### ⚠️ Pin collision warning

The mic uses GPIO38 (DATA) and GPIO39 (CLK), which are the same pins assigned to Internal I²C (SCL=39, SDA=38). You can use **mic OR Internal-I²C but not both.** If you bring up the mic, leave `Wire1` uninitialized (or use a different pin pair).

---

## 1. PDM microphone

```c
// ESP-IDF
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
i2s_new_channel(&chan_cfg, NULL, &rx_chan);

i2s_pdm_rx_config_t pdm_cfg = {
  .clk_cfg = I2S_PDM_RX_CLK_DEFAULT_CONFIG(16000),
  .slot_cfg = I2S_PDM_RX_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO),
  .gpio_cfg = { .clk = GPIO_NUM_39, .din = GPIO_NUM_38 },
};
i2s_channel_init_pdm_rx_mode(rx_chan, &pdm_cfg);
i2s_channel_enable(rx_chan);
```

Copy the Arduino-friendly version from [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp).

---

## 2. LED / Button / Grove

Same patterns as [AtomS3Lite.md](M5AtomS3Lite.md).

---

## Power management

USB-A plug — host-powered. No battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomS3U pin + mic config | [src/M5Unified.cpp:1803](src/M5Unified.cpp#L1803) |
| PDM mic driver | [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) |
