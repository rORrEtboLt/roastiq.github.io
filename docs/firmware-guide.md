# ESP32 Firmware Guide

Build, customize, and update the RoastIQ firmware.

---

## Project Structure

```
Roasting_intelligence/
├── src/
│   └── main.cpp          # ESP32 entry point, WebSocket server, sensor reading
├── include/
│   ├── temperature.h     # isValidTemp(), RollingAverage<N>
│   └── telemetry.h       # buildTelemetryJson()
├── test/                 # Unit tests (run with pio test -e native)
├── platformio.ini        # Build environments
└── docs/                 # Documentation
```

---

## Build Environments

| Environment | Purpose |
|-------------|---------|
| `dev` | Development — verbose logging, erases NVS on flash |
| `esp32doit-devkit-v1` | Production build |
| `native` | Host-based unit tests (no hardware needed) |

---

## Build and Flash

```bash
# Development build
pio run -e dev --target upload --upload-port /dev/ttyUSB0

# Production build
pio run -e esp32doit-devkit-v1 --target upload --upload-port /dev/ttyUSB0

# Monitor serial output
pio device monitor --port /dev/ttyUSB0 --baud 115200
```

---

## How It Works

**Initialization:**
1. Create WiFi access point named `RoastIQ`
2. Start WebSocket server on port 80 at `/ws`
3. Initialize two MAX6675 sensors via SPI

**Main loop (4 Hz):**
1. Read raw BT and ET from MAX6675 sensors
2. Validate temperatures (0.5°C – 1000°C range)
3. Apply 5-sample rolling average for smoothing
4. Send JSON frame via WebSocket

**WebSocket protocol:**

Request from Artisan:
```json
{"id": 1}
```

Response from ESP32:
```json
{"id": 1, "data": {"BT": 185.5, "ET": 210.25}}
```

---

## Unit Tests

Run tests without hardware:

```bash
pio test -e native
```

---

## Downloads

| File | Description |
|------|-------------|
| [main.cpp](https://raw.githubusercontent.com/rORrEtboLt/esp32_temp/master/src/main.cpp) | ESP32 firmware (12 KB) |
| [temperature.h](https://raw.githubusercontent.com/rORrEtboLt/esp32_temp/master/include/temperature.h) | Temperature validation + rolling average |
| [telemetry.h](https://raw.githubusercontent.com/rORrEtboLt/esp32_temp/master/include/telemetry.h) | WebSocket JSON frame builder |
| [platformio.ini](https://raw.githubusercontent.com/rORrEtboLt/esp32_temp/master/platformio.ini) | Build configuration |
| [CIRCUIT.md](https://raw.githubusercontent.com/rORrEtboLt/esp32_temp/master/CIRCUIT.md) | Wiring diagram |

---

## Troubleshooting

**ESP32 won't flash:** Hold the **BOOT** button while pressing **EN** (reset). Release BOOT after upload starts.

**Permission denied on /dev/ttyUSB0:**
```bash
sudo usermod -aG dialout $USER
# Log out and back in
```