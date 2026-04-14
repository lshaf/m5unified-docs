# M5AtomEchoS3R — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomEchoS3R` (145)
**SoC:** ESP32-S3

AtomS3R + **ES8311 audio codec** (I²S playback + mic) + amp controlled by GPIO18 + BMI270/BMM150 IMU. No LCD.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| ES8311 codec (speaker + mic via I²C + I²S) | Yes, I²C `0x18` | raw Wire + ESP-IDF `i2s_std` |
| External class-D amp | Yes, enabled by GPIO18 | digitalWrite |
| BMI270 + BMM150 IMU | Yes (on internal I²C) | as in AtomS3R |
| BtnA | Yes, GPIO41 | digitalRead |
| Grove Port A | Yes | Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| Internal I²C SCL / SDA | 0 / 45 (codec + IMU + mag) |
| Grove (Port A) SCL / SDA | 1 / 2 |
| I²S BCK / WS (shared spk+mic) | 17 / 3 |
| I²S DATA-OUT (speaker) | 48 |
| I²S DATA-IN (mic) | 4 |
| I²S MCK (mic) | 11 |
| **Amp enable** | **18** (HIGH = amp on) |
| BtnA | 41 |

---

## 1. Speaker — full enable sequence

The ES8311 must be configured via I²C **before** I²S data is sent, and the amp must be enabled via GPIO18 after. Callback source: [src/M5Unified.cpp:762 `_speaker_enabled_cb_atom_echos3r`](src/M5Unified.cpp#L762).

Steps:
1. `Wire1.begin(45, 0, 400000)`
2. Write ES8311 enable-bulk (registers `0x00=0x80`, `0x01=0xB5`, `0x02=0x18`, `0x0D=0x01`, `0x12=0x00`, `0x13=0x10`, `0x32=0xFF`, `0x37=0x08`) — full sequence in the callback
3. `digitalWrite(18, HIGH)` → amp on
4. Start I²S transmit on pins BCK=17, WS=3, DOUT=48

```cpp
// Step 2 minimal inline:
uint8_t seq[][2] = {
  {0x00,0x80},{0x01,0xB5},{0x02,0x18},{0x0D,0x01},
  {0x12,0x00},{0x13,0x10},{0x32,0xFF},{0x37,0x08},
};
for (auto& r : seq) {
  Wire1.beginTransmission(0x18);
  Wire1.write(r[0]); Wire1.write(r[1]);
  Wire1.endTransmission();
}
```

To disable: `digitalWrite(18, LOW)` (amp off). ES8311 registers can stay as-is.

---

## 2. Microphone enable sequence

Callback source: [src/M5Unified.cpp:733 `_microphone_enabled_cb_atom_echos3r`](src/M5Unified.cpp#L733).

Different register set on same ES8311 chip:
```
0x00=0x80 0x01=0xBA 0x02=0x18 0x0D=0x01 0x0E=0x02
0x14=0x10 0x17=0xFF 0x1C=0x6A
```
Then start I²S receive on BCK=17, WS=3, DIN=4, MCK=11 (full-duplex with speaker on `I2S_NUM_1`).

Disable writes `0x0D=0xFC`, `0x0E=0x6A`, `0x00=0x00`.

---

## 3. IMU

See [M5AtomS3R.md](M5AtomS3R.md#2-bmi270-imu--bmm150-mag). Identical driver.

---

## 4. Button / Grove

BtnA on GPIO41, Grove on GPIO1/2.

---

## Brightness

No LCD — skip.

## Volume (ES8311 + external amp)

Same dual-layer (software + ES8311 hardware register) as StickS3.

**Software** (default): `magnification = 1`, the callback sets ES8311 reg `0x32 = 0xFF` (**max hardware gain**, unusual — because the speaker is physically small).

```cpp
M5.Speaker.setVolume(128);       // master 0..255
```

**Hardware (ES8311 reg `0x32`)** — each step ≈ 0.5 dB, range -95.5..+12 dB:
```cpp
void es8311_set_vol(uint8_t v) {
  // NOTE: I2C on this board is SDA=45, SCL=0 (internal bus) — Wire1
  Wire1.beginTransmission(0x18);
  Wire1.write(0x32); Wire1.write(v);
  Wire1.endTransmission();
}
// 0dB:  es8311_set_vol(0xBF);
// +6dB: es8311_set_vol(0xDF);   // risky on small speaker — may distort
```

The **amp enable pin GPIO18** is also a hard on/off — `digitalWrite(18, LOW)` silences everything regardless of register values.

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Speaker callback (full register sequence) | [src/M5Unified.cpp:762](src/M5Unified.cpp#L762) |
| Mic callback | [src/M5Unified.cpp:733](src/M5Unified.cpp#L733) |
| AtomEchoS3R pinmap | [src/M5Unified.cpp:1659](src/M5Unified.cpp#L1659) + [:2019](src/M5Unified.cpp#L2019) |
| ES8311 bulk-write helper | [src/M5Unified.cpp:351](src/M5Unified.cpp#L351) |
