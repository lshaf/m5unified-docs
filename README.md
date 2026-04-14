# M5Unified per-device reference (Claude edition)

> One file per board. Dense technical reference for picking the **only** part of
> M5Unified / M5GFX you actually need for a given device.
>
> Generated from source: `src/M5Unified.cpp`, `src/utility/*.{cpp,hpp}`,
> `../M5GFX/src/*`. If something looks wrong, the source wins.

## How to read these files

Every board file has the same sections:

1. **Header** — board enum value, SoC, auto-detect notes.
2. **Component availability matrix** — yes/no per component, at a glance.
3. **Pin map table** — GPIO numbers for internal I²C, ports A/B/C/D/E, SD SPI, RGB LED, power-hold, and anything else that is board-specific.
4. **Display** — panel chip, resolution, bus, pins, reset/backlight control path.
5. **Speaker / Mic** — type (DAC / I²S / PDM / buzzer), pins, sample-rate defaults, which callback M5Unified installs to actually *enable* the chip.
6. **PMIC / Power** — chip (AXP192 / AXP2101 / IP5306 / AW32001 / PY32PMIC / …), I²C address, which rail powers which peripheral, how 5V-out works.
7. **IMU / RTC / Touch / Buttons / LED** — chip, I²C address, GPIO.
8. **Minimal wake-up snippet** — the smallest possible `M5.begin(cfg)` to bring the device up with only what you need.

## Key concepts (read this once)

- **`M5.begin()` vs `M5Unified`**: `M5.begin(cfg)` auto-detects the board and
  wires pins. If you only want one peripheral, **you do not need M5Unified** —
  but you still need the right GPIO numbers, which this doc gives you.
- **PMIC-gated rails**: many M5Stack boards do **not** wire peripherals to a
  regulator; they go through a PMIC (AXP192 / AXP2101). If you don't set the
  right rail, the chip is literally unpowered and will not respond on I²C.
  Every "Turn on" section shows the exact rail write.
- **Enable callbacks**: `M5Unified` installs callbacks on Speaker/Mic that run
  every time `.begin()` is called on them. They do things like flip a PMIC
  GPIO, write ES8311 registers, or toggle a PI4IOE5V6408 I/O-expander pin.
  If you bypass M5Unified, you must replicate the callback yourself.
- **Port A = external I²C**: on nearly every board, Port A is the user-facing
  Grove I²C. "External I²C SCL/SDA" and "Port A pin1/pin2" are the same pins.
- **`_pin_table_*` arrays**: `src/M5Unified.cpp` ~line 65–325 holds the raw per-
  board pin assignments as `{board_id, pin1, pin2, ...}` records. If this doc
  ever disagrees with those tables, the tables are authoritative.

## Conventions

- `—` in a pin table means the pin is not wired on this board (stays `GPIO_NUM_MAX`).
- I²C addresses are shown as `0x__` 7-bit.
- Buttons: `BtnA` / `BtnB` / `BtnC` / `BtnEXT` / `BtnPWR` follow the enum in
  `M5Unified.hpp`. Boards with touch-driven virtual buttons are marked.

## Board index

### ESP32 (original)

- [M5Stack](M5Stack.md) — classic Core (Basic/Gray/Fire)
- [M5StackCore2](M5StackCore2.md)
- [M5Tough](M5Tough.md)
- [M5StickC](M5StickC.md)
- [M5StickCPlus](M5StickCPlus.md)
- [M5StickCPlus2](M5StickCPlus2.md)
- [M5StackCoreInk](M5StackCoreInk.md)
- [M5Paper](M5Paper.md)
- [M5Station](M5Station.md)
- [M5AtomLite](M5AtomLite.md)
- [M5AtomMatrix](M5AtomMatrix.md)
- [M5AtomEcho](M5AtomEcho.md)
- [M5AtomU](M5AtomU.md)
- [M5AtomPsram](M5AtomPsram.md)
- [M5StampPico](M5StampPico.md)
- [M5TimerCam](M5TimerCam.md)

### ESP32-S3

- [M5StackCoreS3](M5StackCoreS3.md)
- [M5StackCoreS3SE](M5StackCoreS3SE.md)
- [M5StickS3](M5StickS3.md)
- [M5AtomS3](M5AtomS3.md)
- [M5AtomS3Lite](M5AtomS3Lite.md)
- [M5AtomS3U](M5AtomS3U.md)
- [M5AtomS3R](M5AtomS3R.md)
- [M5AtomS3RExt](M5AtomS3RExt.md)
- [M5AtomS3RCam](M5AtomS3RCam.md)
- [M5AtomEchoS3R](M5AtomEchoS3R.md)
- [M5StampS3](M5StampS3.md)
- [M5Capsule](M5Capsule.md)
- [M5Dial](M5Dial.md)
- [M5DinMeter](M5DinMeter.md)
- [M5AirQ](M5AirQ.md)
- [M5VAMeter](M5VAMeter.md)
- [M5Cardputer](M5Cardputer.md)
- [M5CardputerADV](M5CardputerADV.md)
- [M5PaperS3](M5PaperS3.md)
- [M5StampPLC](M5StampPLC.md)
- [M5PowerHub](M5PowerHub.md)
- [M5DualKey](M5DualKey.md)

### ESP32-C3

- [M5StampC3](M5StampC3.md)
- [M5StampC3U](M5StampC3U.md)

### ESP32-C6

- [M5NanoC6](M5NanoC6.md)
- [M5UnitC6L](M5UnitC6L.md)
- [ArduinoNessoN1](ArduinoNessoN1.md)

### ESP32-P4

- [M5Tab5](M5Tab5.md)
- [M5UnitPoEP4](M5UnitPoEP4.md)

## Known gaps / not yet documented

- `board_M5CoreMP135` (enum value 20) — enum exists in `board_t` but the
  actual panel/pin definition is not yet present in M5GFX in this repo snapshot.
  Not covered here.
- `board_M5Camera` (enum value 131) — legacy, pinmap only partially wired.
- Accessory flags (`external_speaker.*`, `external_display.*`) are out of scope
  per your request (base-device-only).
