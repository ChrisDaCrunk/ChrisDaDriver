# ChrisDaDriver V4
# WiGLE, WDGWars and Wardrift compatible Wardriving for M5Cardputer

ChrisDaDriver is an ultra-fast, multi-threaded wardriving firmware built specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net) or [wdgwars.pl](https://wdgwars.pl/) and even [wardrift.net](https://wardrift.net).

<img width="826" height="461" alt="bannerwideklein" src="https://github.com/user-attachments/assets/b81b27f2-5060-4acb-8133-d78ff909852b" />

---

## Features

- **WiGLE v1.5 Compliant**: Pre-formatted CSV log files ready for direct WiGLE submission.
- **Parallel Dual-Core Architecture**: 
  - WiFi promiscuous sniffer and channel hopping on **Core 0**.
  - BLE scanning, GPS parsing, UI rendering, and SD writing on **Core 1**.
- **High-Throughput Processing**: Capable of capturing and processing ~22 scan entries per second.
- **Dynamic SD Benchmarking**: Measures write latency on boot to dynamically scale buffer queue limits (`maxSdQueueSize`).
- **Thread-Safe Memory Management**: FreeRTOS mutexes (`dataMutex`) protect shared GPS and scan buffers against race conditions.
- **Flicker-Free Optimized UI**: Direct rendering without heavy canvas overhead to save RAM, featuring a smooth color-transitioning battery indicator (Green > Yellow > Red).
- **Interactive File Management**: Built-in menu to select, upload, or securely delete individual log files directly on the device.
- **Direct API upload to WDGWars**: Upload your csv files directly from the Cardputer to the API.
- 
---

## Speed is everything!
### Wi-Fi Channel Hopping & Loop Times

| Parameter | Value | Details |
| :--- | :--- | :--- |
| **Hop Interval** | `100 ms` | Channel residence time (`WLAN_CHANNEL_HOP_MS = 100`) |
| **Channel Range** | Configurable | Dynamic channel hopping sequence loaded from config |
| **Task Loop Cycle (Core 0)** | `~100 ms` | Base delay (`vTaskDelay(100ms)`) + buffer execution time |

---

### Wi-Fi Scan Routine & Timing

- **Packet Capture (Interrupt):** **Real-time (< 1 ms)** via sniffer callback into ring buffer (size: 32).
- **Buffer Processing:** **~1 ms per packet** (`vTaskDelay(1ms)` per iteration for watchdog feeding).
- **Mutex Timeout (SD Queue):** **Max. 20 ms** wait time for SD queue access.

---

### BLE Scan Times & Overheads (Main Loop Core 1)

- **BLE Scan Window / Interval:** **120 ms** window / **160 ms** interval.
- **BLE Cache Reset:** Every **5.0 s** (clear & restart cycle).
- **Main Loop Cycle:** **~10 ms** base delay + SD flush & UI redraw overhead.
- **Total Logging Throughput:** **~22 log entries / s** (combined Wi-Fi + BLE).

---

## Hardware Requirements

- **M5Stack Cardputer**
- **MicroSD Card** (FAT32 formatted)
- **GPS Module** (connected via Grove or I/O header)

---

## Controls

| Key / Button | Action |
| :--- | :--- |
| **Btn 0 (Short Press)** | Toggle Display Backlighting ON / OFF (Dim / Bright) |
| **Btn 0 (Long Press)** | Open Main Menu |
| **`Enter` (Keyboard)** | Start scanning / Execute selected menu action / Save setting |
| **`Del` (Keyboard)** | Stop scanning / Return or exit menu |
| **`D` (Keyboard)** | Toggle Developer Diagnostic Overlay |

---

## Firmware Architecture

```text
                    ┌────────────────────────┐
                    │  GPS (ATGM336H/CASIC)  │
                    │  UART2 @ 115200 Baud   │
                    └───────────┬────────────┘
                                │ NMEA (PCAS02/03/04)
                                ▼
                    ┌──────────────────────────┐
                    │      TinyGPSPlus         │
                    │  (safeLat, safeLng, ...) │
                    └────────────┬─────────────┘
                                 │ Position + Timestamp
 ┌──────────────────────┐        │        ┌────────────────────────┐
 │ Wi-Fi Sniffer Task   │        │        │ BLE Scan Callbacks     │
 │ (Core 0, Promiscuous)├────────┼────────┤ (Core 1, Active Scan)  │
 └──────────┬───────────┘        │        └───────────┬────────────┘
            │                    │                    │
            └───────────┐        │        ┌───────────┘
                        ▼        ▼        ▼
                   ┌──────────────────────────────┐
                   │    Mutex (dataMutex)         │
                   │ - FlatHashSet (Deduplication)│
                   │ - Memory Protection Threshold│
                   └───────────────┬──────────────┘
                                   │
                                   ▼
                   ┌──────────────────────────────┐
                   │    SD Write Queue            │
                   │   (std::deque<String>)       │
                   └───────────────┬──────────────┘
                                   │
                                   ▼
                   ┌──────────────────────────────┐
                   │    SD Card (WiGLE CSV)       │
                   │    /ChrisDaDriver/WarDrive   │
                   └──────────────────────────────┘

```

---

## Configuration (`config.txt`)

Upon its first boot, **ChrisDaDriver  automatically generates a configuration file at `/ChrisDaDriver/config.txt` on your SD card. You can edit this file to customize pin mappings, scan intervals, and display behavior:

Defaults:
```ini
CONFIG_VERSION V4.0
// ChrisDaDriver Config
// < notes will not be processed

// Logic
AUTO_RUN 1


// GPS Pins & Baud & Config
GPS_BAUD 115200

// 1: GPS / 2: BeiDou / 3: GPS + BeiDou / 4: GLONASS / 5: GPS + GLONASS / 7: GPS + BeiDou + GLONASS
PCAS04 3

// 0: Binary(faster) / 1: NMEA-0183 (standard)
NMEA 0


// M5Stack LoRa Cap GPS Pins
PIN_GPS_RX 15
PIN_GPS_TX 13

// M5Stack GPS V1.1 AT6668 Module to Grove Port
// PIN_GPS_RX 1
// PIN_GPS_TX 2


// SD Pins
SD_CS_PIN 12
SD_MOSI_PIN 14
SD_SCK_PIN 40
SD_MISO_PIN 39


// LoRa Pin
LORA_CS_PIN 5


// Performance & Tuning
BLE_SCAN_INTERVAL 160
BLE_SCAN_WINDOW 120
WLAN_CHANNEL_HOP_MS 100


WLAN_CHANNELS 1,2,3,4,5,6,7,8,9,10,11,12,13


// UI & Button Settings
BTN_LONG_PRESS_TIME_MS 1000
DISPLAY_BRIGHTNESS_LEVEL 150
DISPLAY_BRIGHTNESS_DIMMED 20


WDGWARS_API_Key 0
WARDRIFT_API_Key 0
