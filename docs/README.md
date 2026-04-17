# RoastIQ — Open-Source Coffee Roasting Telemetry

**RoastIQ** is an open-source coffee roasting telemetry system built on the ESP32 microcontroller. It reads temperature data from MAX6675 thermocouples and streams it in real-time to **Artisan Scope** (the industry-standard coffee roasting software) and a web-based dashboard.

---

## Key Features

- **ESP32-based hardware** — affordable, WiFi-enabled, runs locally
- **Artisan Scope compatible** — connects directly via WebSocket or through the optional cloud server
- **4 Hz temperature sampling** — Bean Temperature (BT) and Exhaust Temperature (ET)
- **Web dashboard** — live roast curves, event marking, roast history (requires optional server)
- **Open hardware** — uses off-the-shelf MAX6675 thermocouple amplifiers
- **MIT licensed** — free for commercial and personal use

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RoastIQ System                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐     WebSocket (4 Hz)     ┌──────────────────┐   │
│   │   ESP32      │ ──────────────────────────│  Artisan Scope   │   │
│   │  (RoastIQ)   │                           │  (Desktop App)   │   │
│   │              │                           └──────────────────┘   │
│   │  • MAX6675   │                                                   │
│   │  • WiFi AP   │     REST POST (1 Hz)     ┌──────────────────┐   │
│   │  • WebSocket │ ──────────────────────────│  RoastIQ Server  │   │
│   │    Server    │                           │  (Node.js)       │   │
│   └──────────────┘                           │  • Socket.io    │   │
│                                              │  • Web Dashboard │   │
│                                              └──────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

The ESP32 can operate in two modes:

1. **Standalone (Artisan only)** — ESP32 creates its own WiFi access point. Artisan connects directly. No internet/server required.

2. **Cloud-connected (full system)** — ESP32 sends data to the optional RoastIQ server. Artisan can also poll the server. Enables multi-roaster management and team collaboration.

---

## Hardware Requirements

| Component | Details |
|-----------|---------|
| Microcontroller | ESP32 DOIT DevKit v1 (or compatible) |
| Thermocouples | 2× MAX6675 modules with K-type thermocouples |
| Bean Temperature (BT) | Thermocouple placed in the roasting chamber |
| Exhaust Temperature (ET) | Thermocouple placed in the exhaust duct |
| Power | USB-C or 5V/1A barrel jack |

---

## Quick Start

### 1. Flash the ESP32

```bash
# Install PlatformIO if you haven't
pip install platformio

# Find your USB port
ls /dev/ttyUSB* /dev/ttyACM*

# Flash the firmware (replace /dev/ttyUSB0 with your port)
pio run -e dev --target upload --upload-port /dev/ttyUSB0

# Monitor serial output
pio device monitor --port /dev/ttyUSB0 --baud 115200
```

### 2. Connect to RoastIQ WiFi

On first boot, the ESP32 creates an open WiFi network named **RoastIQ**. Connect from your phone or laptop.

### 3. Configure Artisan Scope

1. Open Artisan → **Config → Device**
2. Set ET/BT device to **WebSocket**
3. Enter URL: `ws://192.168.4.1/ws` (or the IP shown in serial monitor)
4. Press **ON** — BT/ET curves should appear at 4 Hz

See [artisan-integration.md](artisan-integration.md) for detailed setup instructions.

---

## Documentation

- [ESP32 Firmware Deep Dive](esp32-firmware.md) — How `main.cpp` reads sensors, runs the WebSocket server, and communicates with Artisan
- [Artisan Scope Integration](artisan-integration.md) — Direct WebSocket connection vs. server-based connection
- [System Architecture](architecture.md) — Full architecture including the optional Node.js server, Socket.io, and web dashboard
- [RoastIQ vs Artisan Scope](roastiq-vs-artisan.md) — Understanding what each tool does and how they complement each other

---

## Project Structure

```
Roasting_intelligence/
├── src/
│   └── main.cpp          # ESP32 firmware entry point
├── include/
│   ├── temperature.h     # Temperature validation + rolling average
│   └── telemetry.h       # WebSocket JSON frame builder
├── docs/                  # This documentation
├── server/               # Optional Node.js server (Express + Socket.io)
├── web/                  # React web dashboard
├── test/                 # Unit tests for temperature + telemetry
└── platformio.ini        # PlatformIO build configuration
```

---

## License

RoastIQ is open-source under the [MIT License](LICENSE).

---

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.
