# Speaker Volume — cross-device reference

How `M5.Speaker.setVolume(n)` actually works, and how to reach the same result (or go further) without M5Unified.

---

## Two layers

There is **software volume** (DSP scaling in `Speaker_Class`) and **hardware volume** (codec register, on codec boards). M5Unified defaults the hardware register once at enable time and does all dynamic adjustment in software. You can do either or both.

---

## Layer 1 — Software volume (every board)

Source: [src/utility/Speaker_Class.cpp:620](../src/utility/Speaker_Class.cpp#L620).

Effective output per sample:
```
scaled_sample = sample × magnification × (master / 255)² × (channel / 255)²
```

Three parameters:

| Parameter | API | Default | Range |
|-----------|-----|---------|-------|
| `magnification` | `cfg.magnification` in `speaker_config_t`, per-board | varies | 1..255, usually 1..48 |
| `master_volume` | `Speaker.setVolume(n)` | **64** | 0..255 |
| `channel_volume` | `Speaker.setChannelVolume(ch, n)` or `setAllChannelVolume(n)` | **64** per channel | 0..255 |

The quadratic curve maps linear knob movement to perceived loudness.

### Standalone equivalent

```cpp
int   magnification = 16;
uint8_t master = 64;
uint8_t chan   = 64;

float vol = magnification * (master / 255.0f) * (master / 255.0f)
                          * (chan   / 255.0f) * (chan   / 255.0f);
int32_t scaled = int32_t(sample * vol);
// clamp to int16 before writing to I²S:
if (scaled >  32767) scaled =  32767;
if (scaled < -32768) scaled = -32768;
```

If `magnification` is too high, samples will clip (saturate at ±32767) — sounds like raspy distortion. If too low, max volume is too quiet.

### Per-board `magnification` defaults

From [src/M5Unified.cpp](../src/M5Unified.cpp) where each board's `speaker_config_t` is initialized:

| Device | Speaker type | `magnification` | Notes |
|--------|--------------|-----------------|-------|
| M5Stack | DAC (GPIO25) | **8** | 8-bit DAC needs extra gain |
| M5StackCore2 | I²S + PMIC amp | **16** | |
| M5Tough | I²S + PMIC amp | **24** | weaker amp than Core2 |
| M5StickCPlus / CPlus2 | Buzzer (GPIO2) | **48** | |
| M5StackCoreInk | Buzzer (GPIO2) | **48** | |
| M5Dial / DinMeter | Buzzer (GPIO3) | **48** | |
| M5AirQ / VAMeter | Buzzer | **48** | |
| M5Capsule / PaperS3 | Buzzer | **48** | |
| M5StampPLC / UnitC6L | Buzzer | **48** | |
| M5AtomEcho (ESP32) | I²S NS4168 | **12** | |
| Atom + ATOMIC SPK HAT | I²S (no codec) | **16** | |
| Atom + ATOMIC ECHO HAT | ES8311 | **1** | codec does its own gain |
| M5StickS3 | ES8311 | **1** | |
| M5StackCoreS3(SE) | AW88298 | **4** | |
| M5Cardputer | I²S (no codec) | **16** | |
| M5CardputerADV | ES8311 | **16** | hybrid (both paths) |
| M5AtomEchoS3R | ES8311 + amp | **1** | |
| M5Tab5 | ES8388 | **4** | |

### Bumping magnification when the board is too quiet

```cpp
auto cfg = M5.Speaker.config();
cfg.magnification = 32;         // was 16
M5.Speaker.config(cfg);
M5.Speaker.begin();
```

This takes effect on the NEXT `begin()` — changes to `cfg` while the speaker is running have no effect.

---

## Layer 2 — Hardware volume (codec boards only)

Some boards have a real DAC register you can write. M5Unified writes a fixed value at enable time and doesn't touch it at runtime.

### AW88298 — M5StackCoreS3 / CoreS3SE

- I²C addr: `0x36` on internal bus (GPIO11/12)
- Registers are **16-bit** (write hi byte then lo byte)
- Volume reg: `0x0C`
  - high byte: mute (non-zero = muted)
  - low byte: volume step 0..0xFF (0 = silent, 0xFF = full)
- M5Unified sets `0x0064` at enable → unmuted, mid-gain

```cpp
void aw88298_set_vol(uint8_t v /* 0..0xFF */) {
  Wire1.beginTransmission(0x36);
  Wire1.write(0x0C);
  Wire1.write(0x00);   // hi byte = unmuted
  Wire1.write(v);      // lo byte = volume step
  Wire1.endTransmission();
}
aw88298_set_vol(0xFF);   // full hardware gain
aw88298_set_vol(0x40);   // M5Unified default is 0x64
aw88298_set_vol(0x00);   // silent (mute via 0 step)
```

### ES8311 — StickS3 / CardputerADV / AtomEchoS3R / Atom + Atomic Echo HAT

- I²C addr: `0x18` on internal bus
- DAC volume reg: `0x32` (8-bit, LSB = 0.5 dB)
  - `0x00` = mute (-95.5 dB)
  - `0xBF` = **0 dB** (M5Unified default on StickS3 / CardputerADV)
  - `0xFF` = +32 dB **(set on AtomEchoS3R / Atomic Echo — small speaker needs it)**
- Each step = 0.5 dB

```cpp
void es8311_set_vol(uint8_t v) {
  Wire1.beginTransmission(0x18);
  Wire1.write(0x32); Wire1.write(v);
  Wire1.endTransmission();
}
es8311_set_vol(0xBF);   // 0 dB — clean, default
es8311_set_vol(0xDF);   // +16 dB — louder
es8311_set_vol(0xFF);   // +32 dB — max, will distort on large speakers
es8311_set_vol(0x00);   // mute
```

### ES8388 — M5Tab5

- I²C addr: `0x10`
- **Inverted scale**: register value = attenuation
- Regs:
  - `26` (LDACVOL): L channel
  - `27` (RDACVOL): R channel
- `0x00` = 0 dB (loudest), `0xC0` = -96 dB (mute). Each step = -0.5 dB.

```cpp
void es8388_set_att(uint8_t att_half_db /* 0..0xC0 */) {
  Wire1.beginTransmission(0x10);
  Wire1.write(26); Wire1.write(att_half_db);   // L
  Wire1.endTransmission();
  Wire1.beginTransmission(0x10);
  Wire1.write(27); Wire1.write(att_half_db);   // R
  Wire1.endTransmission();
}
es8388_set_att(0x00);    // max volume
es8388_set_att(0x14);    // -10 dB
es8388_set_att(0xC0);    // mute
```

### DAC (M5Stack) — no hardware knob

ESP32's internal DAC is 8-bit. Max amplitude = 255, min = 0. "Volume" = amplitude of the input signal pre-DAC, purely software. Software volume > 200 + magnification 8 = saturates the 8-bit range.

### Buzzers — duty cycle is "volume"

Piezo buzzer: volume = PWM duty cycle. SPL peaks at 50% duty and drops symmetrically.

```cpp
// M5Unified handles this behind the scenes; standalone:
void beep(int freq, int ms, int duty_0_1023) {
  ledcSetup(0, freq, 10);
  ledcAttachPin(buzzer_pin, 0);
  ledcWrite(0, duty_0_1023);
  delay(ms);
  ledcWrite(0, 0);
  ledcDetachPin(buzzer_pin);
}
// beep(2000, 200, 100);   // ~10% duty = quieter
// beep(2000, 200, 512);   // 50% duty = peak SPL
// beep(2000, 200, 900);   // 90% duty = quieter again (past peak)
```

You cannot exceed 50%-duty SPL by cranking harder.

---

## Practical recipes

### Too quiet even at max

1. `M5.Speaker.setVolume(255)` + `setAllChannelVolume(255)` — usually the first fix.
2. Bump `magnification` before `begin()`:
   ```cpp
   auto cfg = M5.Speaker.config();
   cfg.magnification = 64;     // was e.g. 16
   M5.Speaker.config(cfg);
   M5.Speaker.begin();
   ```
3. Codec boards — push the hardware register:
   - ES8311: `0xFF`
   - AW88298: `0xFF`
   - ES8388: `0x00`
4. Check your source amplitude. Int16 samples should peak around ±25000 for loud; weak audio (e.g. ±3000) will always be quiet no matter the volume knob.

### Distorted / clipping

1. First lower magnification (software clipping is the most common cause).
2. Reduce source amplitude.
3. Back off hardware register by -6 dB and raise software instead — software is easier to keep clean.

### Mute quickly without losing config

```cpp
M5.Speaker.setVolume(0);          // software, instant, reversible
```

or on codec boards, mute at hardware level:
```cpp
aw88298_set_vol(0x00);
es8311_set_vol(0x00);
es8388_set_att(0xC0);
```

### Per-channel levels

M5Unified supports 8 virtual channels simultaneously. Useful for mixing music + effects:
```cpp
M5.Speaker.setChannelVolume(0, 200);   // music channel
M5.Speaker.setChannelVolume(1, 80);    // button-click sfx
M5.Speaker.playRaw(music_data, music_len, 44100, false, 1, 0);
M5.Speaker.playRaw(click_wav,  click_len, 44100, false, 1, 1);
```

Combined scaling = master × channel, both quadratic.

---

## Common pitfalls

### "I hear no sound"

1. **PMIC-gated amp boards (Core2, Tough, CoreS3, Tab5, StickS3, AtomEchoS3R):** the amp is **off by default**. You must call `M5.Speaker.begin()` (which runs the enable callback) or replicate the rail/GPIO toggle yourself.
2. `setVolume(0)` or `setAllChannelVolume(0)` accidentally left in.
3. On ESP32 Arduino core 2.x → 3.x migration, the I²S driver changed — old `i2s.h` code won't work; use `ESP_I2S.h`.

### "Speaker and mic aren't full-duplex"

They are on most codec boards (they share the same `i2s_port`). You need to configure both MIC and SPK with the same `i2s_port` number and init them together.

### "Volume jumps between samples"

`setVolume` is applied per-DMA-buffer. Default DMA buf is 256 samples ≈ 5.8 ms at 44.1 kHz — jumps are imperceptible at that rate. If you see clicks, raise `dma_buf_count` or interpolate your own ramp.

### "Setting magnification at runtime doesn't change anything"

Correct — `magnification` is baked into the speaker task at `begin()`. Stop the task, update, restart:
```cpp
M5.Speaker.end();
auto cfg = M5.Speaker.config();
cfg.magnification = 32;
M5.Speaker.config(cfg);
M5.Speaker.begin();
```
