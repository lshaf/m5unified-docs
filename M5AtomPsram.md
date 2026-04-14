# M5AtomPsram — standalone code

**Board enum:** `board_M5AtomPsram` (129)
**SoC:** ESP32-PICO-V3-02 (with 2 MB PSRAM)

Same pinout as [M5AtomLite](M5AtomLite.md); the only functional difference is 2 MB of PSRAM available.

---

## Pin map

Identical to AtomLite:

| Purpose | GPIO |
|---------|------|
| WS2812 | 27 |
| BtnA | 39 |
| Internal I²C SCL / SDA | 21 / 25 |
| Grove (Port A) SCL / SDA | 32 / 26 |

---

## PSRAM

- **Enable in Arduino:** add `-DBOARD_HAS_PSRAM` in `platformio.ini` build_flags, or select "Enabled" in the PSRAM setting.
- **Enable in ESP-IDF:** `idf.py menuconfig` → Component config → ESP32-specific → Support for external PSRAM (`CONFIG_SPIRAM_SUPPORT=y`).

Verify at runtime:
```cpp
Serial.printf("PSRAM: %u bytes free\n", ESP.getFreePsram());   // ~2 MB
```

Allocate in PSRAM explicitly:
```cpp
uint8_t *big = (uint8_t*)ps_malloc(1024 * 1024);  // 1 MB from PSRAM
// or with caps:
uint8_t *big2 = (uint8_t*)heap_caps_malloc(1024 * 1024, MALLOC_CAP_SPIRAM);
```

---

## LED / Button / Grove

Same as [AtomLite.md](M5AtomLite.md).

---

## Power management

USB-powered only. See [M5AtomLite.md § Power management](M5AtomLite.md#power-management).

---

## Source references

| Topic | File : line |
|-------|-------------|
| AtomPsram pin assignment | [src/M5Unified.cpp:1620](src/M5Unified.cpp#L1620) |
