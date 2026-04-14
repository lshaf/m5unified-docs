# M5AtomS3 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomS3` (11)
**SoC:** ESP32-S3 (2 MB PSRAM)

0.85" 128×128 color LCD on the face, single button under the screen, USB-C, Grove port.

> This file shows how to drive each onboard component **without `M5Unified`** — copy only the snippet you need.

---

## What's onboard

| Component | Presence | Standalone library you can use |
|-----------|----------|--------------------------------|
| Display (ST7735 OR GC9107, auto-detected) | Yes | M5GFX (standalone) OR LovyanGFX OR Arduino_GFX |
| Onboard button (under LCD) | Yes | Arduino `digitalRead` |
| Internal I²C bus | Yes (unused on stock AtomS3) | Arduino `Wire1` |
| External I²C (Port A, Grove) | Yes | Arduino `Wire` |
| WS2812 RGB LED | **No** (AtomS3 has LCD, not an LED — that's AtomS3Lite) | — |
| Speaker / Mic / IMU / RTC / SD | **No** | — |

---

## Pin map (full)

| Purpose | GPIO | Notes |
|---------|------|-------|
| LCD MOSI | 21 | SPI3 host |
| LCD SCLK | 17 | SPI3 host |
| LCD MISO | — | not wired (3-wire SPI) |
| LCD CS | 15 | |
| LCD DC | 33 | |
| LCD RST | 34 | |
| LCD BL | 16 | LEDC ch7 PWM, 256 Hz |
| BtnA (under screen) | **41** | active LOW, needs internal pull-up |
| Internal I²C SCL | 39 | |
| Internal I²C SDA | 38 | |
| Grove (Port A) SCL | 1 | yellow wire |
| Grove (Port A) SDA | 2 | white wire |
| Grove 5V / GND | — | direct to USB 5V rail |

### Boot-time detail: GPIO46

`M5Unified::begin()` unconditionally runs:
```cpp
m5gfx::gpio_hi(GPIO_NUM_46);
m5gfx::pinMode(GPIO_NUM_46, m5gfx::pin_mode_t::output);
```
at the top of every ESP32-S3 board (see [src/M5Unified.hpp:339](../src/M5Unified.hpp#L339)). On AtomS3 this pin is **not wired** to anything — the write is a no-op used by Capsule/Dial/DinMeter for their power-hold latch. You can skip it on AtomS3 safely.

---

## 1. Display

### Option A — use M5GFX standalone (easiest; auto-detect handles both ST7735 and GC9107 panel variants)

```cpp
// platformio.ini:  lib_deps = m5stack/M5GFX
#include <M5GFX.h>
M5GFX lcd;

void setup() {
  lcd.init();                  // autodetects board -> configures ST7735/GC9107 + backlight
  lcd.setRotation(0);
  lcd.setBrightness(128);
  lcd.fillScreen(TFT_BLACK);
  lcd.setTextColor(TFT_WHITE);
  lcd.setTextSize(2);
  lcd.drawString("hello", 10, 10);
}
```

That's all — `lcd.init()` runs the autodetect routine in [../M5GFX/src/M5GFX.cpp:1531](../../M5GFX/src/M5GFX.cpp#L1531) which is AtomS3-specific and handles both panel variants.

### Option B — explicit LovyanGFX config (copy-pasteable, no autodetect)

If you want to skip the panel-probe round-trip or use LovyanGFX directly, here is the full config extracted from [M5GFX.cpp:1555](../../M5GFX/src/M5GFX.cpp#L1555):

```cpp
#include <LovyanGFX.hpp>

class LGFX_AtomS3 : public lgfx::LGFX_Device {
  lgfx::Panel_GC9107      _panel;    // use Panel_ST7735S if your unit has ST7735
  lgfx::Bus_SPI           _bus;
  lgfx::Light_PWM         _light;
public:
  LGFX_AtomS3() {
    { auto cfg = _bus.config();
      cfg.spi_host   = SPI3_HOST;
      cfg.spi_mode   = 0;
      cfg.freq_write = 40000000;
      cfg.freq_read  = 16000000;
      cfg.spi_3wire  = true;
      cfg.use_lock   = true;
      cfg.dma_channel = SPI_DMA_CH_AUTO;
      cfg.pin_sclk = 17;
      cfg.pin_mosi = 21;
      cfg.pin_miso = -1;
      cfg.pin_dc   = 33;
      _bus.config(cfg);
      _panel.setBus(&_bus);
    }
    { auto cfg = _panel.config();
      cfg.pin_cs       = 15;
      cfg.pin_rst      = 34;
      cfg.pin_busy     = -1;
      cfg.panel_width  = 128;
      cfg.panel_height = 128;
      cfg.offset_x     = 0;    // GC9107: 0 / ST7735: 2
      cfg.offset_y     = 32;   // GC9107: 32 / ST7735: 31
      cfg.offset_rotation = 0; // GC9107: 0 / ST7735: 2
      cfg.readable     = false;
      cfg.invert       = true; // GC9107 keeps true; ST7735 also true
      cfg.rgb_order    = false;
      cfg.bus_shared   = false;
      _panel.config(cfg);
    }
    { auto cfg = _light.config();
      cfg.pin_bl      = 16;
      cfg.freq        = 256;
      cfg.pwm_channel = 7;
      cfg.invert      = false;
      _light.config(cfg);
      _panel.setLight(&_light);
    }
    setPanel(&_panel);
  }
};

LGFX_AtomS3 lcd;
void setup() { lcd.init(); lcd.setBrightness(128); }
```

If you are unsure which panel your unit has, pick Option A (the probe is ~5 ms). Or do the probe yourself following [M5GFX.cpp:1543](../../M5GFX/src/M5GFX.cpp#L1543).

### Option C — raw Arduino_GFX (smallest dependency)

```cpp
#include <Arduino_GFX_Library.h>
Arduino_DataBus *bus = new Arduino_ESP32SPI(33 /* DC */, 15 /* CS */, 17 /* SCK */, 21 /* MOSI */, -1 /* MISO */, HSPI);
Arduino_GFX *gfx = new Arduino_GC9107(bus, 34 /* RST */, 0 /* rot */, true /* IPS */, 128, 128, 0, 32);

void setup() {
  pinMode(16, OUTPUT);                         // backlight
  ledcSetup(7, 256, 8); ledcAttachPin(16, 7);  // PWM ch7 @256Hz, 8-bit
  ledcWrite(7, 128);                           // 50% brightness

  gfx->begin();
  gfx->fillScreen(BLACK);
  gfx->setTextColor(WHITE);
  gfx->println("hello");
}
```

### Backlight alone (no drawing library)

```cpp
// plain Arduino:
pinMode(16, OUTPUT);
ledcSetup(7, 256, 8);         // channel 7, 256 Hz, 8-bit
ledcAttachPin(16, 7);
ledcWrite(7, 255);            // 0..255 brightness
```

---

## 2. Button (onboard, GPIO41)

The LCD itself is pressable — it clicks GPIO41 to ground.

```cpp
#define BTN_A  41
void setup() {
  pinMode(BTN_A, INPUT_PULLUP);
}
void loop() {
  if (digitalRead(BTN_A) == LOW) {
    // pressed
  }
}
```

For debounce + click/hold detection without M5Unified, copy [src/utility/Button_Class.cpp](../src/utility/Button_Class.cpp) — it's a standalone class that only needs `setRawState(millis(), pressed)`.

---

## 3. Grove Port (external I²C or GPIO)

```cpp
#include <Wire.h>
void setup() {
  Wire.begin(2 /* SDA */, 1 /* SCL */, 400000);
  // scan:
  for (uint8_t a = 1; a < 127; ++a) {
    Wire.beginTransmission(a);
    if (Wire.endTransmission() == 0) Serial.printf("device at 0x%02X\n", a);
  }
}
```

Grove pins are **5 V tolerant inputs** (through level shifters on most M5 Units). The 3.3V and 5V rails on the Grove are always on — there is no "5V output enable" on AtomS3.

### Using Grove as plain GPIO instead

GPIO1 and GPIO2 are regular ADC-capable pins. Use `analogRead()`, `digitalWrite()`, etc.

---

## 4. Internal I²C bus (unused on stock AtomS3)

AtomS3 has no onboard I²C peripherals. The pins exist on the bottom castellations:

```cpp
#include <Wire.h>
Wire1.begin(38 /* SDA */, 39 /* SCL */, 400000);
```

Useful if you solder your own IMU or want to talk to a HAT on the bottom rail.

---

## 5. HAT connector (if you attach Atomic SPK / Echo HAT)

The 8-pin bottom connector exposes GPIO5, 6, 7, 8, 38, 39, + 5V + GND. M5Unified's HAT-detect logic is in [src/M5Unified.cpp:1962](../src/M5Unified.cpp#L1962) (speaker / mic callback installation for S3 Atom family). If you use an Atomic SPK HAT, the I²S pins it wires to are BCK=5, WS=39, DATA=38.

---

## Full minimal standalone sketch

```cpp
#include <M5GFX.h>
#include <Wire.h>

M5GFX lcd;
constexpr int BTN_A = 41;

void setup() {
  // Display
  lcd.init();
  lcd.setBrightness(128);
  lcd.setTextColor(TFT_WHITE);
  lcd.setTextSize(2);
  lcd.drawString("AtomS3", 10, 40);

  // Button
  pinMode(BTN_A, INPUT_PULLUP);

  // Grove port (optional)
  Wire.begin(2, 1, 400000);
}

void loop() {
  static bool last = HIGH;
  bool now = digitalRead(BTN_A);
  if (last == HIGH && now == LOW) {
    lcd.fillScreen(random(0xFFFF));
  }
  last = now;
  delay(10);
}
```

Total dependencies: `M5GFX` only. No `M5Unified`. Flash size: ~600 KB.

---

## Brightness (LCD backlight)

Direct GPIO PWM — **LEDC channel 7, 256 Hz, 9-bit (0..511)**. Duty = `brightness << 1`.

```cpp
constexpr int BL_PIN = 16, BL_CH = 7;
void bl_init() { ledcSetup(BL_CH, 256, 9); ledcAttachPin(BL_PIN, BL_CH); }
void bl_set(uint8_t b) { ledcWrite(BL_CH, b << 1); }
```

With M5GFX: `lcd.setBrightness(0..255)`. `0` = fully off, `1` ≈ barely visible, `255` = max.

To avoid the white-flash-then-dim on boot, set brightness to 0 before `init()`:
```cpp
lcd.setBrightness(0);
lcd.init();
lcd.fillScreen(TFT_BLACK);
lcd.setBrightness(128);
```

## Volume

AtomS3 has no onboard speaker — see [M5AtomEcho.md](M5AtomEcho.md) / [M5AtomEchoS3R.md](M5AtomEchoS3R.md) if you've added a speaker HAT.

---

## Power management

USB-powered only — no battery. `M5.Power.getBatteryLevel()` returns `-2`.

---

## Source references (what to copy out of M5Unified if you need more)

| Topic | File : line | What's there |
|-------|-------------|--------------|
| Board autodetect for AtomS3 | [M5GFX.cpp:1531](../../M5GFX/src/M5GFX.cpp#L1531) | SPI probe logic, ST7735 vs GC9107 discrimination, panel init |
| Backlight driver (PWM) | [lgfx/v1/platforms/esp32/Light_PWM.hpp](../../M5GFX/src/lgfx/v1/platforms/esp32/Light_PWM.hpp) | LEDC-based PWM class |
| Button debounce class | [src/utility/Button_Class.cpp](../src/utility/Button_Class.cpp) | ~300 lines, framework-free |
| I²C helper wrapper | [src/utility/I2C_Class.cpp](../src/utility/I2C_Class.cpp) | thin Wire wrapper with bitOn/bitOff/writeRegister helpers |
