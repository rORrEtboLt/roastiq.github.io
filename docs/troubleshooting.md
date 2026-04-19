# Troubleshooting

Solutions to common issues with RoastIQ.

---

## ESP32 Won't Flash

**Symptom:** Upload fails or hangs

**Fix:** Hold the **BOOT** button on the ESP32 while pressing **EN** (reset). Release BOOT after the upload starts.

---

## Permission Denied on /dev/ttyUSB0

**Symptom:** "Permission denied" error when uploading or monitoring

**Fix:**
```bash
sudo usermod -aG dialout $USER
# Log out and back in for the group change to take effect
```

---

## Artisan Shows "---" Instead of Temperatures

**Symptom:** Connected but no readings appear in Artisan

**Causes and fixes:**

1. **Thermocouple disconnected** — Check serial monitor. If `[TEMP]` shows `0.0°C`, the thermocouple may be loose. Verify connections at the MAX6675 screw terminals.

2. **Wrong network** — Your computer must be connected to the ESP32's network (either the RoastIQ WiFi access point or the same router network).

3. **Wrong IP address** — Verify the WebSocket URL in Artisan matches the IP shown in the serial monitor.

---

## WebSocket Connection Refused

**Symptom:** "Connection refused" error in Artisan

**Fix:**
1. Verify the IP address in Artisan's WebSocket URL
2. Ensure no firewall is blocking port 80
3. Try pinging the ESP32: `ping 192.168.4.1`
4. Check serial output confirms the WebSocket server started

---

## ESP32 Access Point Doesn't Appear

**Symptom:** "RoastIQ" WiFi network not visible after flashing

**Fix:**
1. Hold **BOOT** button and press **EN** (reset)
2. Reflash with the dev environment (erases NVS): `pio run -e dev --target upload`

---

## Temperature Shows 0.0°C

**Symptom:** Serial monitor shows `BT: 0.0°C` or `ET: 0.0°C`

**Cause:** MAX6675 not receiving power or thermocouple not connected

**Fix:**
1. Check 3V3 and GND connections to the MAX6675
2. Ensure the K-type thermocouple is firmly seated in the screw terminals (T+ and T-)
3. Try a known-good thermocouple to rule out a broken probe

---

## Temperature Shows 1023.75°C

**Symptom:** Serial monitor shows unrealistic high temperatures

**Cause:** Thermocouple is disconnected or the T+/T- wires are loose

**Fix:** Check the thermocouple connections at the MAX6675 screw terminals. If the probe is disconnected, the MAX6675 reports this sentinel value.

---

## Temperature Jumps Erratically

**Symptom:** BT/ET values fluctuate wildly

**Cause:** Thermocouple may be cold, connections loose, or leads picking up electrical noise

**Fix:**
1. This is normal for a cold thermocouple — values stabilize as the roaster heats up
2. Check thermocouple connections at the MAX6675 screw terminals
3. Route thermocouple leads away from mains wiring
4. Ensure thermocouples are rated for the temperature range (K-type is good to ~1300°C)

---

## WebSocket Disconnects Frequently

**Symptom:** Artisan loses connection to the ESP32

**Fix:**
1. Reduce the number of WebSocket clients (close other browser tabs connected to the ESP32)
2. Check `pio device monitor` for `[HEAP]` warnings — low memory can cause instability
3. Restart the ESP32 by pressing the EN (reset) button

---

## Further Help

- [Getting Started](getting-started.html) — Initial setup guide
- [Artisan Guide](artisan-guide.html) — Connecting to Artisan
- [Firmware Guide](firmware-guide.html) — ESP32 development