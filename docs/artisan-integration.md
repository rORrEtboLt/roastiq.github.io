---
layout: default
title: Artisan Integration
---

# Artisan Scope Integration

Artisan Scope is the industry-standard desktop application for coffee roast profiling. RoastIQ's ESP32 firmware is designed to **stream live temperature data directly to Artisan**, with no server required.

---

## Two Ways to Connect

### Mode 1: Direct WebSocket (No Server) — Recommended

The ESP32 hosts its own WebSocket server. Artisan connects directly to the ESP32 over your local network.

```
┌──────────────┐         WiFi (local)         ┌──────────────────┐
│   ESP32      │ ◄─────────────────────────── │  Artisan Scope   │
│  (RoastIQ)   │      ws://192.168.x.x/ws     │  (Desktop App)   │
│              │                               └──────────────────┘
│ • WiFi AP    │
│ • WebSocket  │
│   server     │
└──────────────┘
```

**Pros:**
- Works completely offline
- No internet/server required
- Lowest latency (direct connection)

**Cons:**
- Artisan and ESP32 must be on the same network
- No cloud sync, team collaboration, or roast history

### Mode 2: Via RoastIQ Server (Cloud-Connected)

The ESP32 sends data to the optional RoastIQ server. Artisan polls the server's HTTP endpoint.

```
┌──────────────┐   REST POST    ┌──────────────────┐   HTTP Poll   ┌──────────────────┐
│   ESP32      │ ─────────────► │  RoastIQ Server  │ ◄─────────── │  Artisan Scope   │
│  (RoastIQ)   │   (1 Hz)       │  (Node.js)        │   (1 Hz)     │  (Desktop App)  │
└──────────────┘                └──────────────────┘               └──────────────────┘
```

**Pros:**
- Works over the internet (Artisan and ESP32 don't need to be on the same network)
- Cloud roast history, team collaboration
- Multi-roaster management

**Cons:**
- Requires running the RoastIQ server
- Slightly higher latency (HTTP polling vs. direct WebSocket)

---

## Mode 1: Direct WebSocket Setup

### Step 1: Flash and Boot the ESP32

```bash
pio run -e dev --target upload --upload-port /dev/ttyUSB0
pio device monitor --port /dev/ttyUSB0 --baud 115200
```

Note the IP address printed in the serial output:

```
[WiFi] AP "RoastIQ" started — IP: 192.168.4.1
[WS] Artisan endpoint at ws://192.168.4.1/ws
```

### Step 2: Connect Your Device to RoastIQ

Artisan (or your laptop) needs to be on the same network as the ESP32:

- **If Artisan is on the same machine as the ESP32's network:** The ESP32 creates an open AP named `RoastIQ`. Connect your laptop to that WiFi network first.
- **If the ESP32 is on your router WiFi:** Connect your laptop to the same WiFi network. Find the ESP32's IP from your router's DHCP client list, or from the serial output.

### Step 3: Configure Artisan

1. Open **Artisan** → **Config** → **Device**
2. In the **ET/BT** section, click the device dropdown and select **WebSocket**
3. Enter the URL: `ws://192.168.4.1/ws` (or your ESP32's IP address)
4. Click **OK**
5. Press the **ON** button in Artisan's toolbar

You should see BT and ET curves appearing at 4 Hz.

### Serial Output (Expected)

```
[WS] Client #1 connected from 192.168.4.2
[TEMP] BT: 185.5°C  ET: 210.3°C  |  WS clients: 1
[TEMP] BT: 186.1°C  ET: 210.8°C  |  WS clients: 1
...
```

---

## WebSocket Protocol

The ESP32 sends JSON frames at 4 Hz (every 250ms):

```json
{"id":1,"data":{"BT":185.50,"ET":210.25}}
```

| Field | Description |
|-------|-------------|
| `id` | Incrementing message counter |
| `data.BT` | Bean Temperature in Celsius |
| `data.ET` | Exhaust Temperature in Celsius |

Artisan sends **any incoming text message** to request a reading. The ESP32 responds with the latest temperature values.

---

## Mode 2: Server-Based Setup

This requires running the **RoastIQ server** (see `server/` directory).

### ESP32 Configuration

The ESP32 must be configured to send data to the server's telemetry endpoint:

```
POST /api/device/telemetry
Body: { "bt": 185.5, "et": 210.3, "elapsedS": 120 }
```

### Artisan HTTP Background Device

Artisan can poll the server's `/api/artisan/current` endpoint:

1. **Config → Device → ET/BT**
2. Select **HTTP** as the device type
3. URL: `https://your-server.com/api/artisan/current?token=<device_jwt>`
4. Set **T1** (BT) JSON field to `BT`
5. Set **T2** (ET) JSON field to `ET`
6. Poll interval: **1 second**
7. Click **OK** and press **ON**

### Server Response Format

```json
{"BT": 185.5, "ET": 210.3, "etime": 120}
```

### Artisan Phase Alarms

Artisan can POST roast events (CHARGE, FC_START, DROP, etc.) to the server:

```bash
curl -X POST https://your-server.com/api/artisan/event \
  -H "Content-Type: application/json" \
  -d '{"token":"<jwt>","event":"FC_START","BT":195.0,"Time":520}'
```

See `docs/artisan-setup-plan.md` for full details on server-based integration.

---

## Troubleshooting

### "Connection refused" in Artisan

- Verify the IP address is correct (check serial output)
- Ensure no firewall is blocking port 80 on the ESP32 or between networks
- Try pinging the ESP32: `ping 192.168.4.1`

### Artisan shows "---" instead of temperature values

- Confirm Artisan is connected to the same network as the ESP32
- Verify the WebSocket URL in Artisan matches the serial output exactly
- Check the ESP32 serial monitor — if `[TEMP]` shows `0.0°C`, the thermocouple may be disconnected

### ESP32 WebSocket disconnects frequently

- Reduce the number of WebSocket clients (close other browser tabs connected to the ESP32)
- Check `pio device monitor` for `[HEAP]` warnings — low memory can cause instability

### Temperature jumps erratically

- This is normal for a cold thermocouple — values stabilize as the roaster heats up
- Check thermocouple connections at the MAX6675 screw terminals
- Ensure thermocouples are rated for the temperature range (K-type is good to ~1300°C)

### ESP32 AP doesn't appear

- Hold the **BOOT** button and press **EN** (reset) to reboot
- The ESP32 may have previously connected to a WiFi network and not reverted to AP mode — reflash with the `dev` environment which erases NVS: `pio run -e dev --target upload`

---

## Further Reading

- [ESP32 Firmware Deep Dive](esp32-firmware.md) — How the firmware reads sensors and serves WebSocket data
- [System Architecture](architecture.md) — Full system including the optional server
- [RoastIQ vs Artisan Scope](roastiq-vs-artisan.md) — What each tool does
- [Artisan Official Documentation](https://artisan-roasterscope.org/)
