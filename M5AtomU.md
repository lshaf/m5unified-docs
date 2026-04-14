# M5AtomU — standalone code

**Board enum:** `board_M5AtomU` (130)
**SoC:** ESP32-PICO-D4

USB-A plug Atom with PDM microphone only.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| PDM microphone | Yes | ESP-IDF `i2s_pdm_rx` |
| WS2812 LED | Yes, GPIO27 | Adafruit_NeoPixel |
| BtnA (side) | Yes, GPIO39 | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| PDM DATA-IN | 19 |
| PDM CLK (`pin_ws`) | 5 |
| WS2812 | 27 |
| BtnA | 39 |
| Grove SCL / SDA | 32 / 26 |

No PMIC. No speaker. No enable callback on the mic.

---

## 1. PDM microphone (GPIO19 data, GPIO5 clock)

### ESP-IDF snippet

```c
i2s_pdm_rx_config_t pdm_cfg = {
  .clk_cfg = I2S_PDM_RX_CLK_DEFAULT_CONFIG(16000),
  .slot_cfg = I2S_PDM_RX_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO),
  .gpio_cfg = { .clk = GPIO_NUM_5, .din = GPIO_NUM_19, .invert_flags = {0,0,0} },
};
i2s_chan_handle_t rx_chan;
i2s_new_channel(&chan_cfg, NULL, &rx_chan);
i2s_channel_init_pdm_rx_mode(rx_chan, &pdm_cfg);
i2s_channel_enable(rx_chan);
```

Copy the full block from [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) (search for `i2s_pdm_rx`).

---

## 2. LED / Button / Grove

Same as AtomLite.

---

## Power management

USB-A plug — powered by the host port directly. No battery. `M5.Power.getBatteryLevel()` returns `-2`.

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomU pin assignment | [src/M5Unified.cpp:1868](src/M5Unified.cpp#L1868) |
| PDM mic init | [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) |
