# M5AtomEcho — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomEcho` (142)
**SoC:** ESP32-PICO-D4

Voice-assistant Atom: I²S speaker + PDM mic.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| I²S speaker (NS4168 amp) | Yes | ESP-IDF `i2s_std` / Arduino `I2S.h` |
| PDM microphone (SPM1423) | Yes | ESP-IDF `i2s_pdm` |
| WS2812 LED | Yes, GPIO27 | Adafruit_NeoPixel |
| BtnA | Yes, GPIO39 | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| I²S BCK (speaker) | 19 |
| I²S WS (shared speaker + mic CLK) | 33 |
| I²S DATA-OUT (speaker) | 22 |
| PDM DATA-IN (mic) | 23 |
| WS2812 | 27 |
| BtnA | 39 |
| Grove SCL / SDA | 32 / 26 |

**Important:** speaker WS and mic CLK share GPIO33 — they run I²S full-duplex on the same port. If you use speaker and mic, you must init them together on `I2S_NUM_0`.

No PMIC — chips are always powered. No enable callback.

---

## 1. I²S speaker (NS4168 on GPIO22 data out)

### Arduino (ESP32 Arduino core ≥ 3.0, using `ESP_I2S.h`)

```cpp
#include <ESP_I2S.h>
I2SClass i2s;

void setup() {
  i2s.setPins(19 /* BCLK */, 33 /* WS */, 22 /* DOUT */);
  i2s.begin(I2S_MODE_STD, 16000 /* sample rate */, I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO);
  // Write 16-bit PCM samples:
  int16_t sample = 0;
  for (int i = 0; i < 16000; ++i) {
    sample = (i & 0x40) ? 8000 : -8000;   // square wave ~125 Hz
    i2s.write((uint8_t*)&sample, 2);
  }
}
```

### ESP-IDF (`i2s_std` channel)

See [src/utility/Speaker_Class.cpp](src/utility/Speaker_Class.cpp) `_i2s_setup()` for the full channel-config (DMA buffer counts, clock source, slot mode). Pin config boils down to:

```
mclk = -1 (not used)
bclk = 19
ws   = 33
dout = 22
din  = 23   // for mic on same port
```

---

## 2. PDM microphone

```cpp
// ESP-IDF i2s_pdm_rx_mode on I2S_NUM_0
// pin_ws  = 33  (PDM CLK)
// pin_din = 23  (PDM DATA)
// sample_rate = 16000
```

Copy the PDM init from [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) (search for `i2s_pdm`).

---

## 3. LED / Button / Grove

Same as AtomLite — GPIO27 WS2812, GPIO39 BtnA, GPIO26/32 Grove I²C.

---

## Full minimal standalone sketch (speaker tone)

```cpp
#include <ESP_I2S.h>
I2SClass i2s;
int16_t sine[160];  // 100 Hz @ 16 kHz = 160 samples

void setup() {
  for (int i = 0; i < 160; ++i) sine[i] = (int16_t)(6000 * sin(2 * PI * i / 160));
  i2s.setPins(19, 33, 22);
  i2s.begin(I2S_MODE_STD, 16000, I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO);
}

void loop() {
  i2s.write((uint8_t*)sine, sizeof(sine));
}
```

---

## Volume

Software DSP only — the NS4168 amp has **no I²C interface**, no hardware volume register. `magnification = 12` by default.

```cpp
M5.Speaker.setVolume(128);          // master 0..255 (default 64)
M5.Speaker.setAllChannelVolume(128);
```

To bump loudness: `cfg.magnification = 24;` before `begin()`. Too-high values clip.

---

## Power management

USB-powered only — no battery. See [M5AtomLite.md § Power management](M5AtomLite.md#power-management).

---

## Source references

| Topic | File : line |
|-------|-------------|
| I²S speaker init | [src/utility/Speaker_Class.cpp](src/utility/Speaker_Class.cpp) |
| PDM mic init | [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) |
| AtomEcho pin assignment | [src/M5Unified.cpp:2179](src/M5Unified.cpp#L2179) |
