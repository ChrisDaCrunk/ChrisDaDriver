# ChrisDaDriver V2.0 - WiGLE Wardriving for M5Cardputer

ChrisDaDriver is an ultra-fast, multi-threaded wardriving firmware built specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net) or [wdgwars.pl](https://wdgwars.pl/).

<img width="826" height="461" alt="bannerwideklein" src="https://github.com/user-attachments/assets/b81b27f2-5060-4acb-8133-d78ff909852b" />

---

## What's New in V2.0 🚀

- **FlatHashSet Deduplication**: Replaced `std::set` with custom `FlatHashSetUint64` data structures for WLAN and BLE. Fast open-addressing lookup without heap fragmentation.
- **Smart GPS Fallback**: Gracefully handles temporary GPS loss by utilizing cached high-precision coordinates (`safeLat`, `safeLng`) to prevent lost scan entries.
- **Toggleable Developer Overlay**: Press **`D`** on the physical keyboard to show/hide real-time heap usage, SD queue fill level, and GPS stats.
- **Aggressive RAM Protection**: Automatic cache clearing when available RAM drops below 40 KB or capacity limits are reached.
- **Improved UI Elements**: Dedicated visual "Driving" status overlay, smart count scaling (K-units for >10,000 devices), and color-coded GPS status indicators.

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

## Hardware Requirements

- **M5Stack Cardputer**
- **MicroSD Card** (FAT32 formatted)
- **NMEA GPS Module** (ATGM336H / CASIC supported, connected via Port A / Grove)

---

## Controls

| Key / Button | Action |
| :--- | :--- |
| **Btn 0 (G0 / OK)** | Start / Stop Wardriving session (Creates a new CSV log file) |
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
