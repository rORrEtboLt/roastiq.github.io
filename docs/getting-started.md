# Getting Started with RoastIQ

Get your coffee roasting telemetry system running in three steps.

---

## What You Need

| Component | Details |
|-----------|---------|
| Microcontroller | ESP32 DOIT DevKit v1 |
| Temperature sensors | 2x MAX6675 module + K-type thermocouples |
| Power | USB-C cable to ESP32 |
| Computer | macOS, Windows, or Linux |

---

## Step 1: Flash the ESP32

Install PlatformIO and flash the firmware:

```bash
# Install PlatformIO
pip install platformio

# Find your USB port
ls /dev/ttyUSB* /dev/ttyACM*

# Flash the firmware (replace /dev/ttyUSB0 with your port)
pio run -e dev --target upload --upload-port /dev/ttyUSB0

# Monitor serial output
pio device monitor --port /dev/ttyUSB0 --baud 115200
```

The ESP32 will create a WiFi access point named **RoastIQ**. Note the IP address shown in the serial monitor (e.g. `192.168.4.1`).

---

## Step 2: Wire the Thermocouples

Connect two MAX6675 modules to the ESP32:

```
ESP32 DOIT DevKit v1

MAX6675 #1 — Bean Temperature (BT)
  GPIO 5  → SCLK
  GPIO 23 → CS
  GPIO 19 ← SO
  3V3     → VCC
  GND     → GND

MAX6675 #2 — Exhaust Temperature (ET)
  GPIO 26 → SCLK
  GPIO 25 → CS
  GPIO 33 ← SO
  3V3     → VCC
  GND     → GND
```

Pin details:

| Signal | GPIO | Purpose |
|--------|------|---------|
| BT CLK | 5 | Bean Temperature clock |
| BT CS | 23 | Bean Temperature chip select |
| BT DO | 19 | Bean Temperature data output |
| ET CLK | 26 | Exhaust Temperature clock |
| ET CS | 25 | Exhaust Temperature chip select |
| ET DO | 33 | Exhaust Temperature data output |

Connect K-type thermocouples to the T+/T- screw terminals on each MAX6675. Route leads away from mains wiring to reduce noise.

---

## Step 3: Connect to Artisan

Artisan Scope is the standard software for coffee roast profiling.

1. Open **Artisan → Config → Device**
2. Set ET/BT device to **WebSocket**
3. Enter URL: `ws://192.168.4.1/ws`
4. Press **ON**

BT and ET curves appear at 4 Hz.

---

## What's Next?

- [Firmware Guide](firmware-guide.html) — Build, customize, and update the ESP32 firmware
- [Artisan Guide](artisan-guide.html) — Configure Artisan for your roaster
- [Troubleshooting](troubleshooting.html) — Fix common issues
- [Architecture](architecture.html) — How the system works