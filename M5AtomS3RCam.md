# M5AtomS3RCam — standalone code

**Board enum:** `board_M5AtomS3RCam` (144)
**SoC:** ESP32-S3 (8 MB PSRAM)

AtomS3R + **OV-series camera module** (DVP parallel bus). No LCD — LCD pins are repurposed for the camera data lanes. Camera is **not** managed by M5Unified; use Espressif's `esp32-camera`.

---

## Pin map (I²C + button only — camera pins via `esp32-camera`)

| Purpose | GPIO |
|---------|------|
| Internal I²C SCL / SDA | 0 / 45 (IMU + mag + camera SCCB shares this bus) |
| Grove (Port A) SCL / SDA | 1 / 2 |
| BtnA | 41 |

Camera pin assignments (XCLK, PCLK, VSYNC, HSYNC, Y2…Y9) are not in M5Unified — check the AtomS3R-Cam schematic PDF on the M5Stack product page. Typical pattern:
```
XCLK  = dedicated GPIO, normally 20+ MHz
SIOD/SIOC = share I²C with BMI270 + BMM150
VSYNC / HSYNC / PCLK = dedicated
Y2..Y9 = 8 dedicated pins
```

---

## 1. Camera init

```cpp
#include "esp_camera.h"
camera_config_t cfg = { /* fill pins from schematic */ };
cfg.xclk_freq_hz = 20000000;
cfg.pixel_format = PIXFORMAT_JPEG;
cfg.frame_size   = FRAMESIZE_QVGA;
cfg.fb_count     = 2;
cfg.fb_location  = CAMERA_FB_IN_PSRAM;
esp_camera_init(&cfg);
camera_fb_t *fb = esp_camera_fb_get();
// ... use fb->buf (JPEG bytes) ...
esp_camera_fb_return(fb);
```

Because the camera's SCCB wires share the same I²C bus as the BMI270 + BMM150, be careful about I²C speed: SCCB works best at 100 kHz; IMU tolerates 400 kHz. Run the bus at 100 kHz for compatibility, or bit-bang SCCB separately.

---

## 2. IMU

See [M5AtomS3R.md](M5AtomS3R.md#2-bmi270-imu--bmm150-mag). Same driver, same axis fix-up.

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomS3RCam pinmap (shared with S3R) | [src/M5Unified.cpp:1658](src/M5Unified.cpp#L1658) |
| Camera init example | [M5Stack/Atom-CAM-S3R](https://github.com/m5stack) (external) |
