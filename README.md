
# ChrisDaDriver - WiGLE Wardriving for M5Cardputer

ChrisDaDriver is a high-performance, multi-threaded wardriving firmware designed specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net) or https://wdgwars.pl/.

<img width="826" height="461" alt="bannerwideklein" src="https://github.com/user-attachments/assets/b81b27f2-5060-4acb-8133-d78ff909852b" />

---

## Features

- **WiGLE v1.5 Compliant**: Pre-formatted CSV log files ready for direct WiGLE submission.
- **Parallel Dual-Scanning**: 
  - WiFi promiscuous mode sniffer running on **Core 0**.
  - BLE scanner, GPS parser, and UI updating running on **Core 1**.
- **Dynamic SD Card Benchmarking**: Measures write speeds at startup to auto-tune memory queue limits and minimize dropped packets.
- **Thread-Safe Architecture**: Uses FreeRTOS mutexes to handle concurrent data processing reliably.
- **Memory Protection**: Smart memory cleanup and deduplication caching to prevent out-of-memory crashes on long runs.
- **Real-Time Display**: Live count of unique BLE and WiFi devices, current GPS fix status, battery level, and distance traveled.
- **Developer Overlay**: On-screen diagnostic overlay displaying RAM, queue usage, and system time.

---

## Hardware Requirements

- **M5Stack Cardputer**
- **MicroSD Card** (FAT32 formatted)
- **NMEA GPS Module** (connected via Port A / Grove or custom header)

---

## File Output Structure

Log files are stored automatically on the SD card in sequential order:

```text
/ChrisDaDriver/
└── WarDrive/
    ├── wd_000.csv
    ├── wd_001.csv
    └── ...
