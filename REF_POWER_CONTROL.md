# Power Control — cross-device reference

How to turn devices **off**, put them to **sleep**, and **wake** them up. Covers power-hold GPIOs, deep sleep, light sleep, timer sleep, and per-board wake sources.

See [REF_POWER_MANAGEMENT.md](REF_POWER_MANAGEMENT.md) for battery % / charging (different topic).

---

## Two power-down modes

| Mode | Current draw | Wake-up cost | Retains RAM |
|------|--------------|--------------|-------------|
| **powerOff()** | ~µA (batt. only) | full reset | no |
| **deepSleep()** | 10-150 µA | ~500 ms | no (except RTC slow mem) |
| **lightSleep()** | 0.8-2 mA | ~1 ms | yes |
| **timerSleep()** | powerOff + RTC-scheduled wake | full reset | no |

`M5.Power.powerOff()` actually **cuts the 3.3 V rail via the PMIC or power-hold GPIO** on board that supports it — true "off". On boards without a latch, it falls back to deep sleep, which still draws some current.

---

## Power-hold GPIO — which boards have a latch

Boards without a PMIC need the firmware to **keep a GPIO HIGH** to stay powered once the physical power button springs back. Listed by power-hold pin:

| Device | Power-hold GPIO | Notes |
|--------|-----------------|-------|
| M5TimerCam | **33** | assert HIGH to stay on |
| M5StackCoreInk | **12** | assert HIGH + RTC can wake from deep sleep |
| M5Paper | **2** | assert HIGH |
| M5StickCPlus2 | **4** | assert HIGH — also resets charger |
| M5Capsule | **46** | shared S3 boot-hold convention |
| M5Dial / DinMeter / AirQ | **46** | same |
| M5PaperS3 | **44** | toggled-pulse shutdown (see below) |

### Standalone pattern

```cpp
constexpr int POWER_HOLD_PIN = 46;

void setup() {
  pinMode(POWER_HOLD_PIN, OUTPUT);
  digitalWrite(POWER_HOLD_PIN, HIGH);     // LATCH — do this FIRST
  // ... rest of setup ...
}

void shutdown() {
  digitalWrite(POWER_HOLD_PIN, LOW);      // cuts 3V3 regulator
  // board dies instantly if on battery; if USB is plugged in, may stay on
}
```

### PaperS3 quirk — needs PULSE sequence

M5Unified's `powerOff()` does this for PaperS3:
```cpp
for (int i = 0; i < 5; ++i) {
  digitalWrite(44, LOW);  delay(50);
  digitalWrite(44, HIGH); delay(50);
}
```
A single LOW write doesn't latch the shutdown — the charger circuit needs a pulse train.

### Tab5 and NessoN1 — shutdown via I/O expander pulse

```cpp
// Tab5: expander #1 pin 4 pulsed LOW/HIGH 10 times
for (int i = 0; i < 10; ++i) {
  M5.getIOExpander(1).digitalWrite(4, i & 1);
  delay(50);
}

// NessoN1: expander #1 pin 0 pulsed
for (int i = 0; i < 10; ++i) {
  M5.getIOExpander(1).digitalWrite(0, i & 1);
  delay(50);
}
```

---

## `M5.Power.powerOff()` — cut power entirely

```cpp
M5.Power.powerOff();       // does not return (board is off)
```

### What it does internally

[Power_Class.cpp:1193 `powerOff()` → `_powerOff(false)`](../src/utility/Power_Class.cpp#L1193):

1. Per-PMIC shutdown command:
   - **AXP192**: `Axp192.powerOff()` → sets reg 0x32 bit 7 (soft-off)
   - **AXP2101**: `Axp2101.powerOff()` → sets reg 0x10 bit 0
   - **IP5306**: sets BOOST keep-on false so boost auto-shuts
   - **PY32PMIC (M5PM1)**: writes reg `0x0C` = `0x01` → power-off mode
2. If power-hold pin exists: pulse LOW/HIGH 5× (reliably cuts rail on PaperS3-style charger ICs).
3. Special-case Tab5 / NessoN1 / PowerHub I/O-expander pulses.
4. Fall through to `esp_deep_sleep_start()` if the hardware couldn't turn off.

### Standalone per-PMIC

```cpp
// AXP192:
axp192_mod(0x32, 0x80, 0x80);             // bit 7 = power-off bit

// AXP2101:
axp2101_mod(0x10, 0x01, 0x01);             // bit 0 = power-off

// IP5306 — no direct off cmd. Disable boost-keep-on and wait for load to drop:
ip5306_mod(0x00, 0, 0x20);                 // clear bit 5 → auto-off on low load

// PY32PMIC:
pm1_write(0x0C, (pm1_read(0x0C) & ~0x03) | 0x01);
```

---

## `M5.Power.deepSleep(us, touch_wakeup)` — RAM gone, wake on GPIO/timer

```cpp
M5.Power.deepSleep(30 * 1000000);           // 30 sec wake by timer
M5.Power.deepSleep(0, true);                // indefinite, wake by touch
M5.Power.deepSleep(0, false);               // indefinite, wake only by reset
```

### What it does

[Power_Class.cpp:1090](../src/utility/Power_Class.cpp#L1090):
1. Puts panel to sleep (`M5.Display.sleep()`).
2. On IP5306 boards, sets `setPowerBoostKeepOn(true)` so the boost stays on through sleep (prevents M5Stack-battery-base from auto-cutting).
3. If `touch_wakeup` is true AND `_wakeupPin` is configured, enables `esp_sleep_enable_ext0_wakeup(pin, LOW)`.
4. If `micro_seconds > 0`, adds timer wake: `esp_sleep_enable_timer_wakeup(us)`.
5. `esp_deep_sleep_start()` — RAM cleared, CPU halted, only RTC slow memory retained.

### Standalone pattern

```cpp
#include <esp_sleep.h>

void deep_sleep_for(uint64_t us, gpio_num_t wake_pin = GPIO_NUM_MAX) {
  if (us > 0) esp_sleep_enable_timer_wakeup(us);
  if (wake_pin < GPIO_NUM_MAX) {
    esp_sleep_enable_ext0_wakeup(wake_pin, 0);  // 0 = wake on LOW
  }
  esp_deep_sleep_start();
}
```

### RAM survives? Only RTC slow memory

Anything stored in regular RAM is lost. To persist data across deep sleep:
```cpp
RTC_DATA_ATTR int boot_count = 0;        // lives in RTC slow RAM (~8 KB)
void setup() { boot_count++; }
```

---

## `M5.Power.lightSleep()` — RAM retained, ~1 ms wake

Same wake sources as deepSleep but keeps RAM and clocks. ~2 mA vs ~10 µA for deepSleep. Wake from GPIO is near-instant.

```cpp
M5.Power.lightSleep(10 * 1000000);    // light-sleep 10 sec, wake by timer
```

Standalone:
```cpp
if (us > 0) esp_sleep_enable_timer_wakeup(us);
esp_light_sleep_start();               // returns after wake — code continues
```

---

## `M5.Power.timerSleep(rtc_time_t)` — wake at specific time-of-day

Uses the onboard PCF8563 or RX8130 RTC to schedule a wake, then power-offs the board. Only works on boards with a battery-backed RTC and a `_rtcIntPin` configured:

| Device | RTC chip | `_rtcIntPin` (wakes ESP32 on RTC alarm) |
|--------|----------|-----------------------------------------|
| M5StickC / CPlus | PCF8563 | **GPIO35** |
| M5StackCoreInk | PCF8563 | **GPIO19** |
| M5Paper | PCF8563 | — (uses EXT5V trick instead) |
| M5StampPLC | RX8130 | **GPIO14** |
| M5Tab5 | RX8130 | via I/O expander |

**Core2 / Tough / M5Paper / CoreInk caveat** (from M5Unified comments): "can't alarm boot while connected to USB" — AXP192 blocks RTC wake when VBUS is present.

### Usage

```cpp
m5::rtc_time_t t = { 6, 30, 0 };    // wake at 06:30:00
M5.Power.timerSleep(t);              // doesn't return
```

Or:
```cpp
M5.Power.timerSleep(60);             // wake in 60 seconds
```

### Standalone — manually schedule RTC alarm + deep sleep

```cpp
// 1. Set PCF8563 alarm registers (reg 0x09 = minute, 0x0A = hour, 0x0B = day, 0x0C = weekday)
//    Clear bit 7 of each register to enable that field.
// 2. Enable AIE (alarm-interrupt-enable, reg 0x01 bit 1):
pcf8563_mod(0x01, 0x02, 0x02);

// 3. Wire up ESP32 to wake on RTC INT pin:
esp_sleep_enable_ext0_wakeup(GPIO_NUM_35, 0);   // StickC RTC-INT on GPIO35
esp_deep_sleep_start();
```

See [src/utility/rtc/PCF8563_Class.cpp](../src/utility/rtc/PCF8563_Class.cpp) for the full alarm/timer register sequence.

---

## Wake sources per device

`_wakeupPin` is set by M5Unified in [Power_Class.cpp `begin()`](../src/utility/Power_Class.cpp). On `deepSleep(us, touch_wakeup=true)`, this pin is enabled as `ext0_wakeup(pin, LOW)`.

| Device | `_wakeupPin` | Used for |
|--------|--------------|----------|
| M5StackCore2, Tough | **39** | touch panel INT |
| M5StickCPlus2 | **35** | power button |
| M5StackCoreInk | **27** | power button |
| M5Paper | **36** | touch panel INT |
| M5PaperS3 | **48** | touch panel INT |

Boards not in this table either:
- Have no wake pin (use timer only, or rely on PMIC PEK)
- Have wake through the PMIC itself (AXP192/AXP2101 have a wake-on-press feature)

### Add a custom wake pin standalone

```cpp
pinMode(GPIO_NUM_0, INPUT_PULLUP);
esp_sleep_enable_ext0_wakeup(GPIO_NUM_0, 0);      // wake on LOW (BtnA on many boards)
esp_deep_sleep_start();
```

For multiple wake pins, use `ext1_wakeup`:
```cpp
const uint64_t mask = (1ULL << GPIO_NUM_0) | (1ULL << GPIO_NUM_37);
esp_sleep_enable_ext1_wakeup(mask, ESP_EXT1_WAKEUP_ANY_LOW);
esp_deep_sleep_start();
```

---

## Power-key (PWRKEY) handling

On AXP192 / AXP2101 / PY32PMIC boards the physical power button is wired to the PMIC, not to a regular GPIO. M5Unified reads it via:

```cpp
uint8_t state = M5.Power.getKeyState();
// 0 = none, 1 = long pressed, 2 = short clicked, 3 = both
```

### Standalone reads

```cpp
// AXP192:
uint8_t pek = axp192_read(0x46);            // reg 0x46 bit 1 = short, bit 0 = long

// AXP2101:
uint8_t r = axp2101_read(0x49);
uint8_t pek = (r >> 2) & 0x03;              // bit 2 = short, bit 3 = long
if (pek) axp2101_write(0x49, pek << 2);     // write-1-to-clear

// PY32PMIC:
uint8_t state = pm1_read(0x17);             // 0=idle, 2=short, other=hold
```

### Configuring the PEK behavior (AXP192)

Register `0x36`:
- bits 7-6: startup time (128 ms / 512 ms / 1 s / 2 s)
- bits 5-4: long-press time (1 s / 1.5 s / 2 s / 2.5 s)
- bits 3-2: auto-shutdown time (4 s / 6 s / 8 s / 10 s)
- bit 0: PEK short-press wake-up enable

M5Unified sets `0x0C` (128 ms startup, 4 s auto-off).

---

## Special — boards with unique power-off logic

### M5Stack + IP5306 battery base

IP5306 auto-shuts if load < 100 mA for 30 s. Normal deep sleep trips this. Fix **before** sleep:
```cpp
M5.Power.Ip5306.setPowerBoostKeepOn(true);
M5.Power.deepSleep(30 * 1000000);
```

### M5PowerHub — custom I²C command

```cpp
uint8_t buf[6] = {0};
M5.In_I2C.writeRegister(0x50, 0x00, buf, 6, 100000);   // clear all port configs
M5.In_I2C.writeRegister8(0x50, 0xE0, 0x01);             // power-off command
```

---

## Vibration motor (Core2 v1.0)

Core2 v1.0 has a small vibration motor on **AXP192 LDO3**. M5Unified exposes:

```cpp
M5.Power.setVibration(128);         // 0..255 strength
M5.Power.setVibration(0);           // off
```

Standalone:
```cpp
// AXP192 LDO3 voltage 0..3.3V = ~0..max vibration
uint8_t v = (strength * 15) / 255;  // map 0..15 nibble
axp192_mod(0x28, v, 0x0F);           // LDO3 low nibble
axp192_mod(0x12, strength ? 0x08 : 0, 0x08);   // LDO3 enable
```

On Core2 v1.1 (AXP2101), vibration is on **DLDO1** instead.

---

## Common pitfalls

### "Board refuses to turn off"

1. Power-hold pin not being pulled low (forgot to configure it / different pin than expected).
2. USB is plugged in — many boards cannot fully power-off while on USB (the USB-C VBUS holds up the rail after the PMIC shuts down). Test on battery only.
3. AXP192 `powerOff()` writes bit 7 of reg 0x32 but if the shutdown-delay register (0x36 bits 3-2) is long, you'll wait 10 s.

### "Board wakes up immediately from deep sleep"

The `_wakeupPin` is being held LOW at the moment of sleep. Check:
1. Touch panel might be pressed.
2. Button not released properly — add `delay(200)` before entering sleep.
3. External pull-down floating. Use `rtc_gpio_pulldown_en()` on the wake pin:
   ```cpp
   rtc_gpio_pullup_dis((gpio_num_t)wake_pin);
   rtc_gpio_pulldown_en((gpio_num_t)wake_pin);
   ```

### "timerSleep doesn't wake up"

USB plugged in on AXP192 boards (Core2, Tough, CoreInk, Paper) — RTC alarm bypassed while VBUS present. Either:
1. Use on battery only.
2. Switch to `deepSleep(us)` (timer wake via ESP32 internal RTC, not external RTC chip).

### "Current draw in deep sleep is high"

- IP5306 auto-on: `setPowerBoostKeepOn(true)` uses more current (boost stays on). If you want truly minimum sleep current, don't use keep-on, and wake via RTC-chip INT instead.
- Floating GPIOs on deep sleep leak ~10 µA each. `rtc_gpio_isolate()` on all unused pins before sleep.
- Pull-ups on I²C buses: `rtc_gpio_pullup_dis()` on SDA/SCL before sleep.

### "`lightSleep()` unexpectedly wakes on serial"

The USB-CDC/JTAG peripheral generates wake events. Disable Serial before light sleep:
```cpp
Serial.end();
esp_light_sleep_start();
Serial.begin(115200);
```

### "Deep sleep on ESP32-S3 with USB-CDC wakes immediately"

USB-CDC keep-alive packets from the host wake the chip. Either:
1. Unplug USB for true sleep behavior test.
2. Disable USB before sleep: `esp_deep_sleep_disable_rom_logging()` + detach USB.
