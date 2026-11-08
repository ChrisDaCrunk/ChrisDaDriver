# ChrisDaDriver - WiGLE and WDGWars Wardriving for M5Cardputer

ChrisDaDriver is an ultra-fast, multi-threaded wardriving firmware built specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net) or [wdgwars.pl](https://wdgwars.pl/).

<img width="826" height="461" alt="bannerwideklein" src="https://github.com/user-attachments/assets/b81b27f2-5060-4acb-8133-d78ff909852b" />

---

## Release Notes

## What's New in V3.0 🚀

- **Short-Press Display Power Management**: A quick short-press on `BTN0` toggles the display backlighting on and off. When turned off, brightness is dimmed to conserve battery; re-activating instantly restores full operational brightness and redraws the UI without interrupting logging tasks.
- **Dynamic File-Based Configuration System (`config.txt`)**: Automatically scans for `/ChrisDaDriver/config.txt` on the SD card at boot. If missing or outdated, a fresh default config file is generated automatically, allowing seamless customization of hardware pins, scanner timings, and UI settings without reflashing.

---

## Features

- **WiGLE v1.5 Compliant**: Pre-formatted CSV log files ready for direct WiGLE submission.
- **Parallel Dual-Core Architecture**: 
  - WiFi promiscuous sniffer and channel hopping on **Core 0**.
  - BLE scanning, GPS parsing, UI rendering, and SD writing on **Core 1**.
- **High-Throughput Processing**: Capable of capturing and processing ~22 scan entries per second.
- **Dynamic SD Benchmarking**: Measures write latency on boot to dynamically scale buffer queue limits (`maxSdQueueSize`).
- **Thread-Safe Memory Management**: FreeRTOS mutexes (`dataMutex`) protect shared GPS and scan buffers against race conditions.

---

## Speed is everything!
### Wi-Fi Channel Hopping & Loop Times

| Parameter | Value | Details |
| :--- | :--- | :--- |
| **Hop Interval** | `100 ms` | Channel residence time (`WLAN_CHANNEL_HOP_MS = 100`) |
| **Channel Range** | `1 – 13` | Sequential hopping sequence[cite: 5] |
| **Full Sweep Duration** | `~1.3 s` | Total cycle time ($13 \times 100\text{ ms}$) |
| **Task Loop Cycle (Core 0)** | `~100 ms` | Base delay (`vTaskDelay(100ms)`) + 1–5 ms buffer execution time |

---

### Wi-Fi Scan Routine & Timing

* **Packet Capture (Interrupt):** **Real-time (< 1 ms)** via sniffer callback into ring buffer (size: 32).
* **Buffer Processing:** **~1 ms per packet** (`vTaskDelay(1ms)` per iteration for watchdog feeding).
* **Mutex Timeout (SD Queue):** **Max. 100 ms** wait time for SD queue access.

---

### BLE Scan Times & Overheads (Main Loop Core 1)

* **BLE Scan Window / Interval:** **120 ms** window / **160 ms** interval.
* **BLE Cache Reset:** Every **5.0 s** (clear & restart cycle).
* **Main Loop Cycle:** **~10 ms** base delay + SD flush & UI redraw overhead.
* **Total Logging Throughput:** **~22 log entries / s** (combined Wi-Fi + BLE).

---

## Hardware Requirements

- **M5Stack Cardputer**
- **MicroSD Card** (FAT32 formatted)
- **NMEA GPS Module** (ATGM336H / CASIC supported, connected via Port A / Grove)

---

## Controls

| Key / Button | Action |
| :--- | :--- |
| **Btn 0 (Short Press)** | Toggle Display Backlighting ON / OFF |
| **Btn 0 (Long Press)** | Start / Stop Wardriving session (Creates a new CSV log file) |
| **`D` (Keyboard)** | Toggle Developer Diagnostic Overlay |

---

## Firmware Architecture

```text
                     ┌────────────────────────┐
                     │    GPS (ATGM336H/CASIC)│
                     │    UART2 @ 115200 Baud │
                     └───────────┬────────────┘
                                 │ NMEA (PCAS02/03/04)
                                 ▼
                    ┌──────────────────────────┐
                    │      TinyGPSPlus         │
                    │  (safeLat, safeLng, ...) │
                    └────────────┬─────────────┘
                                 │ Position + Timestamp
  ┌──────────────────────┐       │       ┌────────────────────────┐
  │ Wi-Fi Sniffer Task   │       │       │ BLE Scan Callbacks     │
  │ (Core 0, Promiscuous)├───────┼───────┤ (Core 1, Active Scan)  │
  └──────────┬───────────┘       │       └───────────┬────────────┘
             │                   │                   │
             └───────────┐       │       ┌───────────┘
                         ▼       ▼       ▼
                 ┌──────────────────────────────┐
                 │     Mutex (dataMutex)        │
                 │ - FlatHashSet (Deduplication)│
                 │ - Memory Protection Threshold│
                 └───────────────┬──────────────┘
                                 │
                                 ▼
                 ┌──────────────────────────────┐
                 │     SD Write Queue           │
                 │    (std::deque<String>)      │
                 └───────────────┬──────────────┘
                                 │
                                 ▼
                 ┌──────────────────────────────┐
                 │     SD Card (WiGLE CSV)      │
                 │    /ChrisDaDriver/WarDrive   │
                 └──────────────────────────────┘
```

---

## Configuration (`config.txt`)

Upon its first boot, **ChrisDaDriver  automatically generates a configuration file at `/ChrisDaDriver/config.txt` on your SD card. You can edit this file to customize pin mappings, scan intervals, and display behavior:

Example:
```ini
CONFIG_VERSION V3.0
// ChrisDaDriver Config
// < notes will not be processed

// GPS Pins & Baud
PIN_GPS_RX 15
PIN_GPS_TX 13
GPS_BAUD 115200

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

// UI & Button Settings
BTN_LONG_PRESS_TIME_MS 1500
DISPLAY_BRIGHTNESS_LEVEL 80
DISPLAY_BRIGHTNESS_DIMMED 0
