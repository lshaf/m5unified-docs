# M5StickCPlus2 — standalone code

**Board enum:** `board_M5StickCPlus2` (5)
**SoC:** ESP32-PICO-V3-02

Revision of StickCPlus that **drops the AXP192 PMIC** in favor of simpler parts:
- Charger: **AW32001** at I²C `0x49`
- Power hold: **GPIO4** (not a PMIC latch)
- Backlight: **GPIO27 direct PWM** (no PMIC rail)

This makes it much easier to drive standalone — no PMIC init dance.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ST7789 LCD 135×240 | Yes, direct-powered | M5GFX |
| PDM mic | Yes, always-on | ESP-IDF `i2s_pdm_rx` |
| Buzzer | Yes, GPIO2 | ledcWrite |
| MPU6886 IMU | Yes, I²C `0x68` | driver |
| PCF8563 RTC | Yes, I²C `0x51` | driver |
| AW32001 charger IC | Yes, I²C `0x49` | [src/utility/power/AW32001_Class.cpp](src/utility/power/AW32001_Class.cpp) |
| Battery + ADC | Yes, ADC on GPIO38 | analogRead |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold (latch)** | **4** (HIGH = stay on off USB) |
| LCD MOSI / SCLK | 15 / 13 |
| LCD CS / DC / RST | 5 / 14 / 12 |
| LCD BL | **27** (LEDC ch7 PWM @256 Hz — direct GPIO!) |
| PDM DATA-IN | 34 |
| PDM CLK | 0 |
| Buzzer | 2 |
| Red LED / IR TX | 19 |
| Internal I²C SCL / SDA | 22 / 21 |
| Grove (Port A) SCL / SDA | 33 / 32 |
| BtnA / BtnB / **BtnPWR** | 37 / 39 / **35** |
| Battery ADC | 38 |

### ⚠️ Power hold is mandatory

Unlike StickC (AXP192 manages this), StickCPlus2 needs GPIO4 driven HIGH to stay on when USB is unplugged:

```cpp
void setup() {
  pinMode(4, OUTPUT);
  digitalWrite(4, HIGH);    // DO THIS FIRST — otherwise board dies on USB unplug
}
```

Pull it LOW → board shuts off (clean soft-off).

### ⚠️ BtnPWR is plain GPIO

Unlike other Sticks, BtnPWR is on GPIO35 as a regular input — not a PMIC register. Long-press logic you implement yourself (e.g. hold 2 s → `digitalWrite(4, LOW)`).

---

## 1. Display (easy — direct GPIO backlight)

```cpp
#include <M5GFX.h>
M5GFX lcd;
void setup() {
  pinMode(4, OUTPUT); digitalWrite(4, HIGH);   // LATCH
  lcd.init();
  lcd.setBrightness(128);
}
```

No PMIC writes needed. The ST7789 is hardwired to the always-on 3.3 V rail; backlight is plain PWM on GPIO27.

Autodetect: [M5GFX.cpp:837](../../M5GFX/src/M5GFX.cpp#L837).

---

## 2. PDM microphone (always-on)

No LDO0 dance (unlike StickC). Just start I²S PDM RX:
```c
gpio_cfg.clk = GPIO_NUM_0;
gpio_cfg.din = GPIO_NUM_34;
```

---

## 3. AW32001 charger

```cpp
#include <Wire.h>
Wire.begin(21, 22, 400000);
// read charge status (reg 0x0B):
Wire.beginTransmission(0x49); Wire.write(0x0B); Wire.endTransmission(false);
Wire.requestFrom(0x49, 1);
uint8_t stat = Wire.read();
```

Full driver: [src/utility/power/AW32001_Class.cpp](src/utility/power/AW32001_Class.cpp).

---

## 4. Battery voltage

```cpp
float vbat = (analogRead(38) / 4095.0) * 3.3 * 2.0;  // divider ÷2
```

---

## 5. Buzzer / IMU / RTC / Buttons

- Buzzer: GPIO2 PWM.
- IMU: MPU6886 @ `0x68`, internal bus (21/22).
- RTC: PCF8563 @ `0x51`.
- BtnA=37, BtnB=39, BtnPWR=35 (all `INPUT_PULLUP`).

---

## Brightness (LCD backlight)

Direct GPIO PWM — **GPIO27, LEDC channel 7, 256 Hz, 9-bit**.

```cpp
constexpr int BL_PIN = 27, BL_CH = 7;
void bl_init() { ledcSetup(BL_CH, 256, 9); ledcAttachPin(BL_PIN, BL_CH); }
void bl_set(uint8_t b) { ledcWrite(BL_CH, b << 1); }
```

Smooth 256-step control (vs StickCPlus's 13 discrete PMIC steps). Zero = off.

## Volume (buzzer)

Buzzer on GPIO2 — software DSP with `magnification = 48`. Same API as StickCPlus:

```cpp
M5.Speaker.setVolume(128);
```

See [M5StickCPlus.md § Volume](M5StickCPlus.md#volume-buzzer) for duty-cycle caveats.

---

## Power management

Two chips: **AW32001 charger** (I²C `0x49`) + **ADC battery reading** (GPIO38 with 2:1 divider).

### With M5Unified

```cpp
int pct  = M5.Power.getBatteryLevel();            // 0..100 %, derived from voltage
int mv   = M5.Power.getBatteryVoltage();
auto chg = M5.Power.isCharging();                 // from AW32001 status register
M5.Power.setChargeCurrent(200);                   // mA (AW32001 accepts 8..2048 mA)
M5.Power.setChargeVoltage(4200);                  // mV
```

### Standalone — battery voltage via ADC

```cpp
float vbat_mv() {
  // GPIO38, ADC1 ch 2, with 2:1 divider
  int raw = analogRead(38);              // 0..4095
  float mv = (raw * 3300.0f / 4095.0f) * 2.0f;   // × 2 for divider
  return mv;
}

int pct_from_mv(float mv) {
  int level = (mv - 3300) * 100 / (4150 - 3350);   // same formula M5Unified uses
  return (level < 0) ? 0 : (level > 100) ? 100 : level;
}
```

### Standalone — AW32001 charge status

```cpp
// Charge status: reg 0x08, bits 3-4
Wire.begin(21, 22, 400000);
Wire.beginTransmission(0x49); Wire.write(0x08); Wire.endTransmission(false);
Wire.requestFrom(0x49, 1);
uint8_t s = (Wire.read() >> 3) & 0x03;
// 0b00 = not charging, 0b01 = pre-charge, 0b10 = charging, 0b11 = done
bool charging = (s == 0b01 || s == 0b10);

// Charge current (reg 0x02): (reg & 0x3F) * 8 + 8 mA, max 512 mA
// Charge voltage (reg 0x04): (reg & 0x3F) * 15 + 3600 mV, max 4545 mV
```

### GPIO4 power-hold reminder

If GPIO4 goes LOW, the whole board loses power — your `getBatteryLevel()` read will never complete. Always assert GPIO4 HIGH in `setup()` before any other code.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StickCPlus2 autodetect | [M5GFX.cpp:837](../../M5GFX/src/M5GFX.cpp#L837) |
| StickCPlus2 pinmap | [src/M5Unified.cpp:1632](src/M5Unified.cpp#L1632) |
| AW32001 driver | [src/utility/power/AW32001_Class.cpp](src/utility/power/AW32001_Class.cpp) |
