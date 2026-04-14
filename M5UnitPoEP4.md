# M5UnitPoEP4 — standalone code

**Board enum:** `board_M5UnitPoEP4` (148)
**SoC:** ESP32-P4

Power-over-Ethernet unit. No display, no audio. Use case: industrial network node.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| Ethernet PHY (external, RMII) | Yes | ESP-IDF `esp_eth` |
| PoE power input | Yes | N/A |
| BtnA | Yes, GPIO45 | digitalRead |
| Grove Port A | Yes | Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| Internal I²C SCL / SDA | 1 / 0 |
| Grove (Port A) SCL / SDA | 54 / 53 |
| BtnA | 45 |
| Ethernet RMII pins | not managed by M5Unified — configure via `esp_eth_mac_config_t` + `esp_eth_phy_config_t` |

---

## 1. Ethernet init (ESP-IDF `esp_eth`)

```c
esp_netif_init();
esp_event_loop_create_default();
esp_netif_t *eth_netif = esp_netif_new(ESP_NETIF_DEFAULT_ETH());
eth_mac_config_t mac_cfg = ETH_MAC_DEFAULT_CONFIG();
eth_esp32_emac_config_t emac_cfg = ETH_ESP32_EMAC_DEFAULT_CONFIG();
// Set RMII pins per M5UnitPoEP4 schematic
esp_eth_mac_t *mac = esp_eth_mac_new_esp32(&emac_cfg, &mac_cfg);
eth_phy_config_t phy_cfg = ETH_PHY_DEFAULT_CONFIG();
phy_cfg.phy_addr = 1;         // depends on strapping
esp_eth_phy_t *phy = esp_eth_phy_new_ip101(&phy_cfg);  // or lan8720 — check schematic
esp_eth_config_t eth_cfg = ETH_DEFAULT_CONFIG(mac, phy);
esp_eth_handle_t eth;
esp_eth_driver_install(&eth_cfg, &eth);
esp_netif_attach(eth_netif, esp_eth_new_netif_glue(eth));
esp_eth_start(eth);
```

Exact RMII pin assignment is not in M5Unified — check the UnitPoEP4 schematic PDF.

---

## 2. Button

```cpp
pinMode(45, INPUT_PULLUP);
if (digitalRead(45) == LOW) { /* pressed */ }
```

---

## Power management

Powered over Ethernet (PoE) + optional USB. No battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| UnitPoEP4 pinmap | [src/M5Unified.cpp:2597](src/M5Unified.cpp#L2597) |
| Ethernet example | ESP-IDF `examples/ethernet/basic/` |
