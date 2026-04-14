# M5Dial — standalone code

**Board enum:** `board_M5Dial` (12)
**SoC:** ESP32-S3

Round 1.28" GC9A01 touch display with rotary-encoder bezel, RTC, speaker buzzer, battery.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| GC9A01 round LCD (240×240) | Yes | M5GFX OR LovyanGFX `Panel_GC9A01` |
| FT3267 capacitive touch | Yes, I²C `0x38` | LGFX `Touch_FT5x06` (FT3267 compatible) |
| Rotary encoder bezel | Yes | GPIO interrupts |
| Piezo buzzer | Yes, GPIO3 | `ledcWrite` / `tone` |
| PCF8563 RTC | Yes, I²C `0x51` | Adafruit_PCF8563 or raw Wire |
| WS2812 LED | Yes, GPIO21 | Adafruit_NeoPixel |
| Battery + charger (simple) | Yes | read ADC |
| Power-hold latch | GPIO46 | Arduino digitalWrite |

---

## Pin map (full)

| Purpose | GPIO |
|---------|------|
| **Power hold** | **46** (must be HIGH to stay on battery) |
| LCD MOSI / SCLK | 5 / 6 |
| LCD CS / DC / RST | 7 / 4 / 8 |
| LCD BL | 9 (LEDC ch7 PWM @44.1 kHz) |
| Internal I²C SCL / SDA | 12 / 11 (touch, RTC) |
| Grove (Port A) SCL / SDA | 15 / 13 |
| Port B pin1 / pin2 | 1 / 2 |
| Buzzer | 3 |
| WS2812 | 21 |
| Encoder A / B | 40 / 41 |
| Encoder press (BtnA) | 42 |
| Side button (BtnB) | 0 |

### ⚠️ Power-hold is mandatory on battery

Without asserting GPIO46 HIGH **at the very top of `setup()`**, the board will switch off as soon as you release the power button (the button only gives a momentary HIGH to GPIO46, and firmware is expected to latch it).

```cpp
void setup() {
  pinMode(46, OUTPUT);
  digitalWrite(46, HIGH);     // LATCH POWER — do this FIRST
  // ... rest of setup
}
```

---

## 1. Display (GC9A01 round)

### Option A — M5GFX autodetect

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  pinMode(46, OUTPUT); digitalWrite(46, HIGH);
  lcd.init();                  // autodetects Dial, sets panel + touch + BL
  lcd.setBrightness(128);
}
```

Autodetect code: [M5GFX.cpp:1341](../../M5GFX/src/M5GFX.cpp#L1341).

### Option B — explicit LovyanGFX config

```cpp
#include <LovyanGFX.hpp>
class LGFX_Dial : public lgfx::LGFX_Device {
  lgfx::Panel_GC9A01  _panel;
  lgfx::Bus_SPI       _bus;
  lgfx::Light_PWM     _light;
  lgfx::Touch_FT5x06  _touch;
public:
  LGFX_Dial() {
    { auto c = _bus.config();
      c.spi_host = SPI3_HOST; c.spi_mode = 0;
      c.freq_write = 40000000; c.freq_read = 16000000;
      c.spi_3wire = true; c.use_lock = true;
      c.dma_channel = SPI_DMA_CH_AUTO;
      c.pin_sclk = 6; c.pin_mosi = 5; c.pin_miso = -1; c.pin_dc = 4;
      _bus.config(c); _panel.setBus(&_bus); }
    { auto c = _panel.config();
      c.pin_cs = 7; c.pin_rst = 8; c.pin_busy = -1;
      c.panel_width = 240; c.panel_height = 240;
      c.readable = false; c.invert = true; c.bus_shared = false;
      _panel.config(c); }
    { auto c = _light.config();
      c.pin_bl = 9; c.pwm_channel = 7; c.freq = 44100; c.invert = false;
      _light.config(c); _panel.setLight(&_light); }
    { auto c = _touch.config();
      c.x_min = 0; c.x_max = 239; c.y_min = 0; c.y_max = 239;
      c.pin_int = -1; c.bus_shared = false;
      c.i2c_port = 0; c.pin_sda = 11; c.pin_scl = 12; c.i2c_addr = 0x38;
      c.freq = 400000;
      _touch.config(c); _panel.setTouch(&_touch); }
    setPanel(&_panel);
  }
};
LGFX_Dial lcd;
```

---

## 2. Rotary encoder

M5Dial is NOT handled by M5Unified's Button class — encoder must be read separately. Easiest path: `ESP32Encoder`:

```cpp
#include <ESP32Encoder.h>
ESP32Encoder encoder;
void setup() {
  encoder.attachHalfQuad(40, 41);  // A=40, B=41
  encoder.setCount(0);
}
void loop() {
  int pos = encoder.getCount();    // increments/decrements on rotation
}
```

Encoder press is a plain button on GPIO42 (active LOW).

---

## 3. Buzzer (GPIO3)

```cpp
void tone_ms(int freq, int ms) {
  ledcSetup(0, freq, 10);
  ledcAttachPin(3, 0);
  ledcWrite(0, 512);       // 50% duty
  delay(ms);
  ledcWrite(0, 0);
  ledcDetachPin(3);
}
```

---

## 4. PCF8563 RTC (I²C `0x51`)

```cpp
#include <Wire.h>
void setup() {
  Wire1.begin(11 /* SDA */, 12 /* SCL */, 400000);
}
// Read time:
struct tm t;
Wire1.beginTransmission(0x51); Wire1.write(0x02); Wire1.endTransmission(false);
Wire1.requestFrom(0x51, 7);
// bytes are BCD: seconds/minutes/hours/day/weekday/month/year
```

Full driver in [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp).

---

## 5. Buttons

| Button | GPIO | Active |
|--------|------|--------|
| BtnA (encoder press) | 42 | LOW |
| BtnB (side) | 0 | LOW (⚠️ strapping) |

---

## Minimal standalone sketch

```cpp
#include <M5GFX.h>
#include <ESP32Encoder.h>
M5GFX lcd;
ESP32Encoder enc;

void setup() {
  pinMode(46, OUTPUT); digitalWrite(46, HIGH);   // LATCH POWER
  lcd.init(); lcd.setBrightness(128);
  enc.attachHalfQuad(40, 41);
  pinMode(42, INPUT_PULLUP);
}

void loop() {
  lcd.setCursor(60, 100);
  lcd.printf("%d   ", (int)enc.getCount());
  if (digitalRead(42) == LOW) enc.setCount(0);
  delay(30);
}
```

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO9, LEDC ch7, 44.1 kHz, 9-bit**.

```cpp
void bl_init() { ledcSetup(7, 44100, 9); ledcAttachPin(9, 7); }
void bl_set(uint8_t b) { ledcWrite(7, b << 1); }
```

44.1 kHz avoids audible whine from the buzzer.

## Volume (buzzer)

GPIO3 buzzer, `magnification = 48`. See [M5StickCPlus.md § Volume](M5StickCPlus.md#volume-buzzer).

```cpp
M5.Speaker.setVolume(128);
```

---

## Power management

Simple charger + ADC battery read. M5Unified doesn't explicitly wire a `batAdcCh` for Dial (check [src/utility/Power_Class.cpp](../src/utility/Power_Class.cpp) around line 220 for the S3 table), but `getBatteryLevel()` works through a shared S3-path.

Practically:
```cpp
int pct = M5.Power.getBatteryLevel();       // 0..100
int mv  = M5.Power.getBatteryVoltage();     // mV (through PMIC-less path)
auto chg = M5.Power.isCharging();            // charge_unknown
```

Shutdown: pull GPIO46 LOW.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Dial autodetect | [M5GFX.cpp:1341](../../M5GFX/src/M5GFX.cpp#L1341) |
| Dial pinmap | [src/M5Unified.cpp:1680](src/M5Unified.cpp#L1680) |
| PCF8563 driver | [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp) |
