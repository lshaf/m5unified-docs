# M5StackCoreInk — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StackCoreInk` (6)
**SoC:** ESP32-D0WDQ6

E-paper Core form factor. 1.54" GDEW0154 panel with a buzzer, RTC, 5-button layout, and direct GPIO power-hold.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| GDEW0154D67 or GDEW0154M09 EPD (200×200) | Yes | M5GFX |
| Buzzer | Yes, GPIO2 | ledcWrite |
| PCF8563 RTC | Yes, I²C `0x51` | PCF8563 driver |
| Battery + charger (simple) | Yes | ADC on GPIO35 |
| Power hold | GPIO12 | digitalWrite |

No speaker (beyond the buzzer), no mic, no IMU, no SD, no touch, no PMIC.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold** | **12** (HIGH = stay on) |
| EPD MOSI / MISO / SCLK | 23 / 34 / 18 |
| EPD CS / DC / RST | 9 / 15 / 0 |
| EPD BUSY | 4 |
| Internal I²C SCL / SDA | 22 / 21 (RTC) |
| Grove (Port A) SCL / SDA | 33 / 32 |
| Buzzer | 2 |
| Battery ADC | 35 |
| BtnA / BtnB / BtnC (face) | 37 / 38 / 39 |
| **BtnEXT (top rotary click)** | 5 |
| **BtnPWR (side)** | 27 |

### ⚠️ GPIO0 = EPD RST = strapping pin

CoreInk wires the EPD RST to GPIO0. Holding GPIO0 LOW during boot puts the ESP32 in download mode. If your code manipulates GPIO0 early, make sure it's HIGH before SPI starts.

---

## 1. Power hold (mandatory)

```cpp
void setup() {
  pinMode(12, OUTPUT);
  digitalWrite(12, HIGH);
}
```

Without this, the board powers off ~1 second after the physical power switch springs back.

---

## 2. E-paper display

```cpp
#include <M5GFX.h>
M5GFX epd;
void setup() {
  digitalWrite(12, HIGH); pinMode(12, OUTPUT);
  epd.init();
  epd.setEpdMode(epd_mode_t::epd_fast);
}
```

Autodetect: [M5GFX.cpp:784](../../M5GFX/src/M5GFX.cpp#L784). Two panel variants (D67 and M09) are distinguished by probe ID.

### Refresh cadence

Full refresh ≈ 2 s; fast refresh ≈ 400 ms. Don't refresh faster than ~1 Hz unless using `epd_fastest` (which introduces ghosting).

---

## 3. Buzzer (GPIO2)

```cpp
void beep(int ms) {
  ledcSetup(0, 4000, 10);
  ledcAttachPin(2, 0);
  ledcWrite(0, 512);
  delay(ms);
  ledcDetachPin(2);
}
```

---

## 4. RTC (PCF8563 @ `0x51`)

Can wake the ESP32 from deep sleep via its INT pin — commonly used for scheduled-wake e-paper clocks.

```cpp
// set alarm register then enter deep sleep; RTC INT is wired to ESP32's RTC GPIO.
```

Full PCF8563 alarm + INT setup: [src/utility/rtc/PCF8563_Class.cpp](src/utility/rtc/PCF8563_Class.cpp).

---

## 5. Buttons (5 buttons — unique to CoreInk)

```cpp
pinMode(37, INPUT_PULLUP);   // BtnA
pinMode(38, INPUT_PULLUP);   // BtnB
pinMode(39, INPUT_PULLUP);   // BtnC
pinMode(5, INPUT_PULLUP);    // BtnEXT (top rotary click)
pinMode(27, INPUT_PULLUP);   // BtnPWR (side)
```

Long-press BtnPWR for soft-off: read GPIO27, if held > 2s do `digitalWrite(12, LOW)`.

---

## Brightness

E-paper — **no backlight**. `setBrightness()` is a no-op. To make the image "darker" use `epd_quality` mode (16 gray levels) or invert pixels.

## Volume (buzzer)

GPIO2 buzzer, `magnification = 48`. Same as other buzzer boards:

```cpp
M5.Speaker.setVolume(128);
M5.Speaker.tone(2000, 200);
```

If using HAT SPK instead (external amp on the side connector), M5Unified installs `_speaker_enabled_cb_hat_spk` which drives GPIO25 HIGH to enable the amp. Volume is still software DSP, `magnification = 32` in the HAT path.

Standalone buzzer:
```cpp
void beep(int freq, int ms, int duty_0_1023 = 512) {
  ledcSetup(0, freq, 10); ledcAttachPin(2, 0);
  ledcWrite(0, duty_0_1023);
  delay(ms);
  ledcDetachPin(2);
}
// quieter:  beep(2000, 200, 100);
// louder:   beep(2000, 200, 512);   // 50% duty = peak SPL
```

---

## Power management

No PMIC — just a simple charger IC + ADC battery read.

### With M5Unified

```cpp
int pct = M5.Power.getBatteryLevel();     // 0..100 (from ADC, lags)
int mv  = M5.Power.getBatteryVoltage();   // mV
auto chg = M5.Power.isCharging();          // charge_unknown (no dedicated status pin)
```

### Standalone battery voltage (ADC on GPIO35)

CoreInk uses a **~5:1 voltage divider** (non-standard) — ratio 25.1 / 5.1 ≈ 4.922:

```cpp
float vbat_mv() {
  int raw = analogRead(35);
  float mv_adc = raw * 3300.0f / 4095.0f;
  return mv_adc * 4.922f;            // recover real battery mV
}

int pct_from_mv(float mv) {
  int pct = (mv - 3300) * 100 / (4150 - 3350);
  return (pct < 0) ? 0 : (pct > 100) ? 100 : pct;
}
```

### Charging detection

There is **no charge-status GPIO** on CoreInk — standalone, you can't read it directly. Indirect methods:
1. Sample voltage over ~30 s and check slope (+0.5 mV/s ≈ charging).
2. Check if USB is plugged in (there's no VBUS pin exposed either — you'd need an external voltage divider).

### Power-off

Pull **GPIO12 LOW** to shut down on battery:
```cpp
digitalWrite(12, LOW);        // immediate soft-off
```
M5Unified's `M5.Power.powerOff()` does exactly this.

---

## Source references

| Topic | File : line |
|-------|-------------|
| CoreInk autodetect | [M5GFX.cpp:784](../../M5GFX/src/M5GFX.cpp#L784) |
| CoreInk pinmap | [src/M5Unified.cpp:1601](src/M5Unified.cpp#L1601) |
| GDEW0154 panel drivers | [M5GFX/src/lgfx/v1/panel/Panel_GDEW0154D67.hpp](../../M5GFX/src/lgfx/v1/panel/Panel_GDEW0154D67.hpp), [Panel_GDEW0154M09.hpp](../../M5GFX/src/lgfx/v1/panel/Panel_GDEW0154M09.hpp) |
