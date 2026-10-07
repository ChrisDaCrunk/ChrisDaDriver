# ChrisDaDriver - WiGLE and WDGWars Wardriving for M5Cardputer

ChrisDaDriver is an ultra-fast, multi-threaded wardriving firmware built specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net) or [wdgwars.pl](https://wdgwars.pl/).

<img width="826" height="461" alt="bannerwideklein" src="https://github.com/user-attachments/assets/b81b27f2-5060-4acb-8133-d78ff909852b" />

---

## Release Notes

### What's New in V2.5 🚀

- **Full NMEA Telemetry Activation**: Explicitly enabled all standard NMEA sentences (`GGA`, `GLL`, `GSA`, `GSV`, `RMC`, `VTG`, `ZDA`) via hardware configuration (`PCAS03`) for comprehensive satellite diagnostics and improved tracking reliability.
- **Fix-Gated Packet Processing**: Deferred all Wi-Fi and BLE beacon processing until a valid GPS fix is established. Eliminates heap exhaustion and prevents unexpected device reboots caused by high BLE device density during initial boot/GPS search.
- **"Waiting for GPS" Visual Indicator**: Added a clear status box overlay to the display during the GPS acquisition phase, providing immediate visual feedback before data collection starts.

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
