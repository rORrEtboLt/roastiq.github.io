# Connecting to Artisan Scope

Artisan Scope is the industry-standard desktop app for coffee roast profiling. RoastIQ streams live temperature data directly to Artisan.

---

## Two Ways to Connect

### Mode 1: Direct WebSocket (No Server)

The ESP32 hosts its own WebSocket server. Artisan connects directly over your local network. **No internet required.**

```
ESP32 (RoastIQ) ←── WiFi ───→ Artisan Scope
ws://192.168.4.1/ws
```

**Pros:** Works completely offline, lowest latency
**Cons:** No cloud sync, no team collaboration

### Mode 2: Via Server

The ESP32 sends data to the RoastIQ server. Artisan polls the server's HTTP endpoint.

**Pros:** Works over the internet, cloud roast history
**Cons:** Requires running the RoastIQ server

---

## Mode 1: Direct Connection

### Step 1: Get the ESP32 IP Address

After flashing, the serial monitor shows:
```
[WiFi] AP "RoastIQ" started — IP: 192.168.4.1
[WS] Artisan endpoint at ws://192.168.4.1/ws
```

### Step 2: Connect Artisan

1. Connect your computer to the **RoastIQ** WiFi network (or to the same network as the ESP32)
2. Open **Artisan → Config → Device**
3. Set ET/BT device to **WebSocket**
4. Enter URL: `ws://192.168.4.1/ws`
5. Press **ON**

BT and ET curves appear at 4 Hz.

---

## Troubleshooting

**Artisan shows "---" instead of temperatures:**
- Confirm your computer is connected to the ESP32's network
- Verify the WebSocket URL matches the serial output
- Check the ESP32 serial monitor — if `[TEMP]` shows `0.0°C`, the thermocouple may be disconnected

**Connection refused:**
- Verify the IP address is correct
- Ensure no firewall is blocking port 80
- Try pinging the ESP32: `ping 192.168.4.1`

**ESP32 AP doesn't appear:**
- Hold the **BOOT** button and press **EN** (reset) to reboot
- Reflash with: `pio run -e dev --target upload` (erases NVS)

**Temperature jumps erratically:**
- Normal for a cold thermocouple — values stabilize as the roaster heats up
- Check thermocouple connections at the MAX6675 screw terminals

---

## Further Reading

- [Getting Started](getting-started.html) — Flash the ESP32 and wire thermocouples
- [Troubleshooting](troubleshooting.html) — Fix common issues