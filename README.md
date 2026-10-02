# ChrisDaDriver - WiGLE Wardriving for M5Cardputer

ChrisDaDriver is a high-performance, multi-threaded wardriving firmware designed specifically for the **M5Cardputer**. It captures WiFi networks (802.11 b/g/n) and Bluetooth Low Energy (BLE) devices in parallel, enriches the data with GPS coordinates, and exports logs in the official **WiGLE CSV format (v1.5)** for seamless upload to [wigle.net](https://wigle.net).

---

## Features

- **WiGLE v1.5 Compliant**: Pre-formatted CSV log files ready for direct WiGLE submission[cite: 2].
- **Parallel Dual-Scanning**: 
  - WiFi promiscuous mode sniffer running on **Core 0**[cite: 1, 3].
  - BLE scanner, GPS parser, and UI updating running on **Core 1**.
- **Dynamic SD Card Benchmarking**: Measures write speeds at startup to auto-tune memory queue limits and minimize dropped packets.
- **Thread-Safe Architecture**: Uses FreeRTOS mutexes to handle concurrent data processing reliably.
- **Memory Protection**: Smart memory cleanup and deduplication caching to prevent out-of-memory crashes on long runs[cite: 1, 2].
- **Real-Time Display**: Live count of unique BLE and WiFi devices, current GPS fix status, battery level, and distance traveled.
- **Developer Overlay**: On-screen diagnostic overlay displaying RAM, queue usage, and system time.

---

## Hardware Requirements

- **M5Stack Cardputer**
- **MicroSD Card** (FAT32 formatted)[cite: 1]
- **NMEA GPS Module** (connected via Port A / Grove or custom header)[cite: 1]

### Default Pin Mapping

| Peripheral | Pin (ESP32-S3) |
| :--- | :--- |
| GPS RX | Pin 15[cite: 1] |
| GPS TX | Pin 13[cite: 1] |
| GPS Baudrate | 115200 baud[cite: 1] |
| SD Card CS | Pin 12[cite: 1] |
| SD Card MOSI | Pin 14[cite: 1] |
| SD Card MISO | Pin 39[cite: 1] |
| SD Card SCK | Pin 40[cite: 1] |

---

## File Output Structure

Log files are stored automatically on the SD card in sequential order:

```text
/ChrisDaDriver/
└── WarDrive/
    ├── wd_000.csv
    ├── wd_001.csv
    └── ...
