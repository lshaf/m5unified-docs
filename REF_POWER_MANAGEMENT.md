# Power Management — cross-device reference

How to read **battery percentage**, **voltage**, **current**, and **charging state** on every M5 device — via `M5.Power.*` and standalone register reads.

See [REF_POWER_CONTROL.md](REF_POWER_CONTROL.md) for power on/off, sleep, and wake-source control (different topic).

---

## PMIC chip matrix

| Chip | I²C addr | Battery % source | Devices |
|------|----------|------------------|---------|
| **AXP192** | `0x34` | voltage-derived | StickC, StickCPlus, Core2 v1.0, Tough, Station |
| **AXP2101** | `0x34` | **coulomb-counted** (reg `0xA4` direct) | Core2 v1.1, CoreS3, CoreS3SE |
| **IP5306** | `0x75` | 5 discrete levels (top nibble of reg 0x78) | M5Stack w/battery base |
| **AW32001** | `0x49` | no gauge — voltage via ADC (GPIO38) | StickCPlus2 |
| **PY32PMIC / M5PM1** | `0x6E` | voltage via PM1 ADC | StickS3 |
| **BQ27220** | `0x55` | standard TI fuel gauge | some ESP32-C6 variants |
| **INA226** | `0x40`/`0x41` | bus voltage × multiplier (for 2S Li-Po) | Tab5, VAMeter, StampPLC |
| **Plain ADC + charger** | — | voltage via MCU ADC + divider | TimerCam, CoreInk, Paper, PaperS3, Capsule, Dial, DinMeter, AirQ, Cardputer, CardputerADV |
| **USB only, no battery** | — | N/A | Atom*, Stamp*, NanoC6, DualKey, UnitPoE, StampPLC, UnitC6L, NessoN1 |

---

## `M5.Power.*` API (works regardless of chip)

```cpp
int  pct  = M5.Power.getBatteryLevel();     // 0..100, -1 err, -2 unsupported
int  mv   = M5.Power.getBatteryVoltage();   // mV, -1 err
int  cur  = M5.Power.getBatteryCurrent();   // mA signed (+ charge / − discharge)
int  vbus = M5.Power.getVBUSVoltage();      // USB rail mV
auto chg  = M5.Power.isCharging();          // is_charging / is_discharging / charge_unknown
M5.Power.setBatteryCharge(true);            // enable/disable charging
M5.Power.setChargeCurrent(500);             // target charge current mA
M5.Power.setChargeVoltage(4200);            // end-of-charge voltage mV
M5.Power.getType();                          // pmic_axp192, pmic_axp2101, pmic_ip5306, ...
```

### `is_charging_t` enum

```cpp
enum is_charging_t {
  is_discharging = 0,
  is_charging,
  charge_unknown
};
```

`charge_unknown` = the board has no hardware signal to tell. Most plain-ADC boards (Cardputer, Dial, DinMeter, AirQ, TimerCam, CoreInk, Paper) return this. You can implement your own heuristic by looking at voltage trend.

---

## Per-PMIC implementation

### AXP192 (StickC, StickCPlus, Core2 v1.0, Tough, Station)

I²C `0x34` on internal bus.

#### Battery voltage
Register 0x78..0x79 — 12-bit, LSB = 1.1 mV.
```cpp
uint8_t hi = axp192_read(0x78);
uint8_t lo = axp192_read(0x79) & 0x0F;
uint16_t raw = (hi << 4) | lo;
float vbat = raw * 1.1f / 1000.0f;     // volts
```

#### Battery %
Derived from voltage — same piecewise formula M5Unified uses:
```cpp
int pct;
if      (raw > 3150) pct = (raw - 3075) * 0.16f;   // 3150..4150 mV → 12..100%
else if (raw > 2690) pct = (raw - 2690) * 0.027f;  // 2690..3150 mV → 0..12%
else                 pct = 0;
if (pct > 100) pct = 100;
```

#### Charging bit
```cpp
bool charging = axp192_read(0x00) & 0x04;
```

#### Charge current (actual, into battery)
Register 0x7A..0x7B, 13-bit, LSB = 0.5 mA.
```cpp
uint16_t raw_c = (axp192_read(0x7A) << 5) | (axp192_read(0x7B) & 0x1F);
float mA = raw_c * 0.5f;
```

#### Discharge current (out of battery)
Register 0x7C..0x7D, same format.

#### VBUS voltage
Register 0x5A..0x5B, 12-bit, LSB = 1.7 mV.
```cpp
uint16_t raw_v = (axp192_read(0x5A) << 4) | (axp192_read(0x5B) & 0x0F);
float vbus = raw_v * 1.7f / 1000.0f;   // volts
```

#### Set charge current (max 1320 mA)
Register 0x33, low 4 bits = current step (100..1320 mA in 80 mA steps after +100).
```cpp
// charge current = 100 + step*80 mA; step 0..15
uint8_t step = (max_mA - 100) / 80;
uint8_t reg = axp192_read(0x33);
axp192_write(0x33, (reg & 0xF0) | (step & 0x0F));
```

---

### AXP2101 (Core2 v1.1, CoreS3, CoreS3SE) — has hardware fuel gauge

I²C `0x34`. **Battery % is a direct register read** — no guessing.

#### Battery %
```cpp
uint8_t pct = axp2101_read(0xA4);   // 0..100 directly
```

#### Battery voltage
Register 0x34..0x35, 14-bit, LSB = 1 mV.
```cpp
uint16_t raw = ((axp2101_read(0x34) & 0x3F) << 8) | axp2101_read(0x35);
float vbat = raw / 1000.0f;   // volts
```

#### Charging state (more reliable than AXP192 bit)
Register 0x01 bits 5-6:
- `0b01` (0x20) = charging
- `0b10` (0x40) = discharging
- `0b00` = standby

```cpp
uint8_t s = axp2101_read(0x01);
uint8_t chg_state = (s >> 5) & 0b11;
bool charging    = (chg_state == 0b01);
bool discharging = (chg_state == 0b10);
```

#### VBUS present + voltage
```cpp
bool vbus_ok = axp2101_read(0x00) & 0x20;   // bit 5
// voltage from reg 0x38..0x39, 14-bit, LSB = 1 mV
uint16_t raw_v = ((axp2101_read(0x38) & 0x3F) << 8) | axp2101_read(0x39);
float vbus_v = raw_v / 1000.0f;
```

#### Internal temperature
Register 0x3C..0x3D.
```cpp
uint16_t raw_t = (axp2101_read(0x3C) << 8) | axp2101_read(0x3D);
float tempC = 22.0f + (7274 - raw_t) / 20.0f;
```

#### Power key press
Register 0x49 bits 2-3 — write-1-to-clear.
```cpp
uint8_t pek = (axp2101_read(0x49) >> 2) & 0x03;
// 1 = short click, 2 = long press, 3 = both
if (pek) axp2101_write(0x49, pek << 2);  // clear
```

#### Battery present
```cpp
bool bat_inserted = axp2101_read(0x00) & 0x08;   // bit 3
```

---

### IP5306 (M5Stack with battery base) — 5-level gauge only

I²C `0x75`. Very limited interface.

#### Battery %
Register `0x78` — top nibble encodes 5 discrete levels.
```cpp
uint8_t r = ip5306_read(0x78);
int pct;
switch (r >> 4) {
  case 0x00: pct = 100; break;
  case 0x08: pct = 75;  break;
  case 0x0C: pct = 50;  break;
  case 0x0E: pct = 25;  break;
  default:   pct = 0;   break;
}
```

#### Charging
```cpp
bool charging = ip5306_read(0x70) & 0x08;
```

#### Set charge current
Register 0x24, bits 0-4 (0..31), step = 100 mA + 50 mA base.
```cpp
// max_mA 50..3150:
uint8_t step = (max_mA > 50) ? (max_mA - 50) / 100 : 0;
if (step > 31) step = 31;
uint8_t reg = ip5306_read(0x24);
ip5306_write(0x24, (reg & 0xE0) | step);
```

#### Boost auto-off disable (critical on M5Stack)

IP5306 auto-powers-off if load < ~100 mA for 30 s. M5Stack in deep-sleep or light-sleep will trip this and the board dies. Fix:
```cpp
// SYS_CTL0 reg 0x00 bit 5 = "boost keep-on even at low load"
uint8_t ctl = ip5306_read(0x00);
ip5306_write(0x00, ctl | 0x20);
```

This is what `M5.Power.Ip5306.setPowerBoostKeepOn(true)` does.

---

### AW32001 (StickCPlus2) — charger only, no gauge

I²C `0x49`. Does charging — **no battery fuel gauge**. Battery voltage is read from a separate ADC on GPIO38.

#### Charge status
Register 0x08, bits 3-4:
- `0b00` = not charging
- `0b01` = pre-charge (low battery)
- `0b10` = charging
- `0b11` = charge done

```cpp
uint8_t r = aw32001_read(0x08);
uint8_t s = (r >> 3) & 0x03;
bool charging = (s == 0b01 || s == 0b10);
bool done     = (s == 0b11);
```

#### Battery voltage via ADC (StickCPlus2 specific)
GPIO38 with 2:1 divider:
```cpp
float vbat_mv = (analogRead(38) * 3300.0f / 4095.0f) * 2.0f;
```

#### Set charge current
Register 0x02, bits 0-5: `(reg & 0x3F) × 8 + 8 mA` → 8..512 mA.
```cpp
uint8_t step = (max_mA - 8) / 8;  if (step > 0x3F) step = 0x3F;
uint8_t reg = aw32001_read(0x02);
aw32001_write(0x02, (reg & 0xC0) | step);
```

#### Set charge voltage
Register 0x04, bits 0-5: `(reg & 0x3F) × 15 + 3600 mV` → 3.6..4.545 V.

---

### PY32PMIC / M5PM1 (StickS3) — custom firmware on a PY32 micro

I²C `0x6E` on internal bus (SDA=47, SCL=48). Not a standard chip — register map is M5-specific. Full driver: [src/utility/power/PY32PMIC_Class.cpp](../src/utility/power/PY32PMIC_Class.cpp).

#### Charging status
Register `0x12` bit 0 — **LOW = charging** (inverted from expected):
```cpp
bool charging = !(pm1_read(0x12) & 0x01);
```

#### Battery voltage
PM1 performs an ADC read on its own GPIO0, accessible over I²C. Exact protocol is multi-step — easiest is to copy `PY32PMIC_Class.cpp` verbatim.

#### Power key state
```cpp
uint8_t state = pm1_read(0x17);
// 0 = no press, 2 = short click, 1 or other = hold
```

#### Power off
```cpp
// reg 0x0C bits 1:0 = 01 → power off
uint8_t r = pm1_read(0x0C);
r = (r & ~0x03) | 0x01;
pm1_write(0x0C, r);
```

---

### BQ27220 (some C6 variants, NessoN1 base)

Standard Texas Instruments fuel gauge. I²C `0x55`.

#### Battery voltage
Register 0x08..0x09 (little-endian):
```cpp
uint16_t mv = bq27220_read(0x08) | (bq27220_read(0x09) << 8);
float vbat = mv / 1000.0f;
```

#### State of charge (%)
Register 0x2C..0x2D (little-endian):
```cpp
uint16_t pct = bq27220_read(0x2C) | (bq27220_read(0x2D) << 8);
```

Full driver: [src/utility/power/BQ27220_Class.cpp](../src/utility/power/BQ27220_Class.cpp).

---

### INA226 on 2S Li-Po (Tab5)

Tab5 has two Li-Po cells in series (nominal 7.4 V). The INA226 on internal bus at `0x41` measures the total pack voltage.

```cpp
uint16_t ina226_read(uint8_t reg) {
  Wire1.beginTransmission(0x41); Wire1.write(reg); Wire1.endTransmission(false);
  Wire1.requestFrom(0x41, 2);
  return (Wire1.read() << 8) | Wire1.read();
}
float pack_V  = ina226_read(0x02) * 0.00125f;    // total 2S volts
float cell_mV = pack_V * 500.0f;                  // per-cell mV (÷ 2 × 1000)
```

Battery % uses the standard per-cell curve `(cell_mV - 3300) × 100 / 850`.

Charge status pin is on I/O expander `0x43` GPIO6 (LOW = charging):
```cpp
bool charging = !(pi4io_read(0x43, 0x0F) & 0x40);
```

---

### Plain ADC boards

No gauge chip — battery voltage is read via a resistor divider on a specific GPIO, then M5Unified applies the standard 3300-4150 mV → 0-100% linear map.

| Device | ADC GPIO | Divider ratio | Notes |
|--------|----------|---------------|-------|
| M5TimerCam | 38 | **1.513** | unusual ratio |
| M5StackCoreInk | 35 | **4.92** (= 25.1 / 5.1) | 5:1 divider |
| M5Paper | 35 | 2.0 | standard 2:1 |
| M5PaperS3 | 3 | 2.0 | + GPIO4 CHG_STAT pin |
| M5Capsule | 6 | 2.0 | |
| M5AirQ | 14 (ADC2) | 2.0 | |
| M5DinMeter | 10 | 2.0 | |
| M5Cardputer | 10 | 2.0 | |
| M5CardputerADV | 10 | 2.0 | |
| M5StickCPlus2 | 38 | 2.0 | |

### Standalone formula (all ADC boards)

```cpp
float vbat_mv(int adc_pin, float ratio) {
  int raw = analogRead(adc_pin);             // 0..4095
  float mv_adc = raw * 3300.0f / 4095.0f;    // ADC reading in mV
  return mv_adc * ratio;                      // actual battery mV
}

int pct_from_mv(float mv) {
  int pct = (mv - 3300) * 100 / (4150 - 3350);
  return (pct < 0) ? 0 : (pct > 100) ? 100 : pct;
}
```

### ADC calibration accuracy

ESP32's ADC is noisy (±50 mV typical). For better results:
1. Use `analogReadMilliVolts(pin)` instead of `analogRead(pin)` (applies internal calibration).
2. Sample 16× and average.
3. Use `esp_adc_cal_characterize()` and `esp_adc_cal_raw_to_voltage()` for per-chip calibration curve.

---

## Boards with dedicated charge-status pin (standalone `isCharging()`)

| Device | Pin | Active level |
|--------|-----|--------------|
| M5PaperS3 | GPIO4 | LOW = charging |
| M5StickS3 | PM1 reg 0x12 bit 0 | LOW = charging |
| M5Tab5 | PI4IO #1 pin 6 | LOW = charging |
| M5PowerHub | (indirect — `battery_current < -10 mA`) | — |
| All AXP192 | reg 0x00 bit 2 | HIGH = charging |
| All AXP2101 | reg 0x01 bits 5-6 | `01` = charging |
| All AW32001 | reg 0x08 bits 3-4 | `01`/`10` = charging |
| All IP5306 | reg 0x70 bit 3 | HIGH = charging |

All other ADC-only boards return `charge_unknown`.

---

## 5V output to Grove / Port A

Some PMIC-gated boards have a software-controllable 5V rail on Port A:

| Device | How to enable 5V out |
|--------|----------------------|
| M5StickC, StickCPlus, Core2 v1.0, Tough | `M5.Power.setExtOutput(true)` → AXP192 EXTEN bit (reg 0x12 bit 6) |
| M5StackCore2 v1.1, CoreS3 | `M5.Power.setExtOutput(true)` → AXP2101 BLDO2 |
| M5Station | Per-port via `setExtOutput(true, ext_port_mask)` |
| Others | 5V is always on (directly from USB) |

Standalone AXP192:
```cpp
axp192_mod(0x12, 0x40, 0x40);     // EXTEN = 1 → 5V on Port A
```

Standalone AXP2101:
```cpp
// enable BLDO2 at 3300 mV:
axp2101_write(0x95, (3300 - 500) / 100);   // BLDO2 voltage
axp2101_mod(0x90, 0x20, 0x20);              // BLDO2 enable bit
```

---

## Common pitfalls

### "Battery % jumps around"

Expected on voltage-derived gauges (AXP192, IP5306, ADC-only, PM1). Under load the voltage sags; when load drops, voltage rises again — so reading during high current draw shows lower %. Workarounds:
- Average 10-20 samples.
- Use AXP2101 boards if you need accuracy (it has a coulomb counter).

### "Battery 100% drops to 80% after a few minutes"

Newly-charged Li-Po surface charge dissipates. Real charge state is lower than the open-circuit voltage initially suggests. Wait 1-2 min after unplugging charger for voltage to settle.

### "isCharging() always returns charge_unknown"

You're on an ADC-only board (Cardputer, Dial, DinMeter, AirQ, TimerCam, CoreInk, Paper, VAMeter, Capsule). There's no hardware charge-status line — M5Unified can't tell. Workaround: monitor voltage trend over ~30 s; `+0.5 mV/s` ≈ charging.

### "getBatteryCurrent() returns 0 on my AXP2101 board"

AXP2101's `getBatteryChargeCurrent()` and `getBatteryDischargeCurrent()` are stubbed in the current driver (always return 0.0f). The chip does expose current but it requires different registers — not yet implemented.

### "setChargeCurrent(X) doesn't work on M5Stack"

The IP5306 on early M5Stack battery bases is in I²C-disabled mode (reads ignored). You can only change charge current via hardware strapping. `setChargeCurrent()` silently no-ops.
