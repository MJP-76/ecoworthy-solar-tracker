# ESP32 & ESP8266 Support for tracker_bridge

This document describes the modifications made to the `tracker_bridge` ESPHome component to support both ESP8266 and ESP32 platforms.

## Overview

The original `tracker_bridge` component was designed for ESP8266 only. This update adds ESP32 support while maintaining full backward compatibility with ESP8266.

## Changes Made

### 1. Platform-Specific ESP-NOW Includes

```cpp
#if defined(ARDUINO_ARCH_ESP32)
#include <esp_now.h>
#elif defined(ARDUINO_ARCH_ESP8266)
extern "C" {
#include <espnow.h>
}
#endif
```

### 2. ESP-NOW Initialization

ESP32 uses `esp_err_t` error handling, while ESP8266 uses simple return codes:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
esp_err_t err = esp_now_init();
if (err != ESP_OK) {
  ESP_LOGE(TAG, "esp_now_init failed: %s", esp_err_to_name(err));
  return;
}
#elif defined(ARDUINO_ARCH_ESP8266)
if (esp_now_init() != 0) {
  ESP_LOGE(TAG, "esp_now_init failed");
  return;
}
esp_now_set_self_role(ESP_NOW_ROLE_COMBO);
#endif
```

### 3. ESP-NOW Callback Signature

ESP32 uses `const uint8_t *` and `int len`, while ESP8266 uses `uint8_t *` and `uint8_t len`:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
static void recv_cb_(const uint8_t *mac, const uint8_t *data, int len);
#elif defined(ARDUINO_ARCH_ESP8266)
static void IRAM_ATTR recv_cb_(uint8_t *mac, uint8_t *data, uint8_t len);
#endif
```

### 4. ESP-NOW Peer Registration

ESP32 uses `esp_now_peer_info_t` struct, while ESP8266 uses direct function call:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
esp_now_peer_info_t peer_info = {};
memcpy(peer_info.peer_addr, bcast, 6);
peer_info.channel = mesh_channel_;
peer_info.encrypt = false;
esp_now_add_peer(&peer_info);
#elif defined(ARDUINO_ARCH_ESP8266)
esp_now_add_peer(bcast, ESP_NOW_ROLE_SLAVE, mesh_channel_, NULL, 0);
#endif
```

### 5. WiFi RSSI Reading

ESP32 uses `esp_wifi_sta_get_ap_info()`, while ESP8266 uses `WiFi.RSSI()`:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
wifi_ap_record_t ap_info;
int8_t rssi = -127;
if (esp_wifi_sta_get_ap_info(&ap_info) == ESP_OK) {
  rssi = ap_info.rssi;
}
#elif defined(ARDUINO_ARCH_ESP8266)
int8_t rssi = (int8_t)WiFi.RSSI();
#endif
```

### 6. WiFi Connectivity Check

ESP32 uses `esp_wifi_sta_get_ap_info()`, while ESP8266 uses `WiFi.isConnected()`:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
wifi_ap_record_t ap_info;
if (esp_wifi_sta_get_ap_info(&ap_info) != ESP_OK) return false;
#elif defined(ARDUINO_ARCH_ESP8266)
if (!WiFi.isConnected()) return false;
#endif
```

### 7. MAC Address Reading

ESP32 uses `esp_read_mac()`, while ESP8266 uses `WiFi.macAddress()`:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
esp_err_t mac_err = esp_read_mac(my_mac_, ESP_MAC_WIFI_STA);
if (mac_err != ESP_OK) {
  ESP_LOGE(TAG, "esp_efuse_mac_get_default failed: %s", esp_err_to_name(mac_err));
  memset(my_mac_, 0, 6);
}
#elif defined(ARDUINO_ARCH_ESP8266)
WiFi.macAddress(my_mac_);
#endif
```

## Wiring

### ESP32 (ESP-WROOM-32)

| ESP32 | Tracker |
|---|---|
| 5V (or 3.3V) | Pad 3 (+) |
| GND | Pad 4 (-) |
| GPIO16 (RX2) | Pin 17 (IR receiver signal leg) |

**No pull-up resistor needed.** ESP32 UART handles it natively.

### ESP8266 (D1 Mini Pro / NodeMCU)

| ESP8266 | Tracker |
|---|---|
| 5V | Pad 3 (+) |
| GND | Pad 4 (-) |
| TX (GPIO1) + RX (GPIO3) soldered together | Pin 17 (IR receiver signal leg) |
| 10kΩ pull-up from TX+RX junction to 3.3V | |

**Important:** GPIO1 (TX) is a boot pin on ESP8266. The connection to pin 17 may interfere with boot. Solutions:
- Use a 5-second boot delay in the ESPHome config
- Hot-plug the signal wire after boot
- Use a stronger pull-up (4.7kΩ or 2.2kΩ)
- Use ESP32 instead (recommended)

## ESPHome Configuration

### ESP32 Config (`solar-tracker.yaml`)

```yaml
esp32:
  board: esp32dev
  framework:
    type: arduino

uart:
  id: uart_bus
  rx_buffer_size: 512
  baud_rate: 9600
  tx_pin: GPIO17  # UART2 TX
  rx_pin: GPIO16  # UART2 RX
```

### ESP8266 Config (`solar-tracker-3.yaml`)

```yaml
esp8266:
  board: d1_mini_pro

esphome:
  on_boot:
    - delay: 5s
    - lambda: |-
        ESP_LOGI("boot", "Boot delay complete, tracker_bridge starting");

uart:
  id: uart_bus
  rx_buffer_size: 512
  baud_rate: 9600
  tx_pin: GPIO1
  rx_pin: GPIO3
```

## Manual Jog Buttons

The following template buttons can be added for manual panel control:

```yaml
button:
  - platform: template
    name: "Jog North"
    icon: mdi:arrow-up-bold
    on_press:
      - lambda: |-
          id(bridge).write_str_frame_("!jog ax=0 dir=+ ms=2000");
  - platform: template
    name: "Jog South"
    icon: mdi:arrow-down-bold
    on_press:
      - lambda: |-
          id(bridge).write_str_frame_("!jog ax=0 dir=- ms=2000");
  - platform: template
    name: "Jog East"
    icon: mdi:arrow-right-bold
    on_press:
      - lambda: |-
          id(bridge).write_str_frame_("!jog ax=1 dir=+ ms=2000");
  - platform: template
    name: "Jog West"
    icon: mdi:arrow-left-bold
    on_press:
      - lambda: |-
          id(bridge).write_str_frame_("!jog ax=1 dir=- ms=2000");
```

## Files Modified

- `esphome/components/tracker_bridge/tracker_bridge.h` — Added `#ifdef` guards for ESP32/ESP8266 platform-specific code

## Files Added

- `ESP32_SUPPORT.md` — This documentation file
- `esphome/solar-tracker.yaml` — ESP32 configuration example
- `esphome/solar-tracker-3.yaml` — ESP8266 configuration example with boot delay

## Compatibility

- **ESP32:** Full support, no pull-up resistor needed, no boot issues
- **ESP8266:** Full support with pull-up resistor and boot delay
- **Original ESP-01S:** Unchanged, still works as before

## Merging with Upstream

These changes are designed to be merged with the original `jtubb/ecoworthy-solar-tracker` repository. All platform-specific code is wrapped in `#ifdef` guards, so the original ESP8266 behavior is preserved when compiling for ESP8266.

To merge:
1. Fork the original repository
2. Copy the modified `tracker_bridge.h` to `esphome/components/tracker_bridge/`
3. Copy the example YAML files to `esphome/`
4. Copy this documentation to the repo root
5. Submit a pull request to the original repository
