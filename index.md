---
layout: default
title: RoastIQ
---

# RoastIQ — Open-Source Coffee Roasting Telemetry

**RoastIQ** is an open-source coffee roasting telemetry system built on the ESP32 microcontroller. Stream real-time Bean Temperature (BT) and Exhaust Temperature (ET) to **Artisan Scope** and a web dashboard.

## Key Features

- **ESP32-based hardware** — affordable, WiFi-enabled, runs locally
- **Artisan Scope compatible** — connects directly via WebSocket
- **4 Hz temperature sampling** — Bean Temperature (BT) and Exhaust Temperature (ET)
- **Web dashboard** — live roast curves, event marking, roast history
- **Open hardware** — uses off-the-shelf MAX6675 thermocouple amplifiers
- **MIT licensed** — free for commercial and personal use

## Architecture

```
ESP32 (RoastIQ) ──── WebSocket (4 Hz) ────► Artisan Scope (desktop)
         │
         └──── REST POST (1 Hz) ────► RoastIQ Server ────► Web Dashboard
```

## Quick Start

1. Flash the ESP32 with PlatformIO
2. Connect Artisan to `ws://<esp32-ip>/ws`
3. Start roasting!

## Documentation

- [ESP32 Firmware Deep Dive](docs/esp32-firmware.html) — How main.cpp reads sensors and serves WebSocket data
- [Artisan Scope Integration](docs/artisan-integration.html) — Direct WebSocket and server-based connection modes
- [System Architecture](docs/architecture.html) — Full system architecture
- [RoastIQ vs Artisan Scope](docs/roastiq-vs-artisan.html) — Understanding what each tool does

## License

RoastIQ is open-source under the [MIT License](LICENSE).
