# ESP32 Firmware Deep Dive

This document explains how the ESP32 firmware in `src/main.cpp` works — from reading thermocouple temperatures to serving data over WebSocket to Artisan Scope.

---

## Hardware Setup

### ESP32 Board

The firmware targets the **ESP32 DOIT DevKit v1** (or any ESP32 with sufficient GPIO). It uses the Arduino framework via PlatformIO.

### MAX6675 Thermocouple Amplifiers

The system uses two **MAX6675** thermocouple amplifier modules, each connected via SPI:

| Signal | GPIO | Purpose |
|--------|------|---------|
| BT_DO  | 19   | Bean Temperature — SPI data output |
| BT_CS  | 23   | Bean Temperature — chip select |
| BT_CLK | 5    | Bean Temperature — SPI clock |
| ET_DO  | 33   | Exhaust Temperature — SPI data output |
| ET_CS  | 25   | Exhaust Temperature — chip select |
| ET_CLK | 26   | Exhaust Temperature — SPI clock |

```
ESP32                MAX6675 (BT)           MAX6675 (ET)
───────              ───────────            ───────────
GPIO 5  ──────────── CLK                   CLK
GPIO 23 ──────────── CS                    CS
GPIO 19 ──────────── SO                   SO
                       │                     │
                   K-type               K-type
                 thermocouple         thermocouple
```

The MAX6675 is a cold-junction-compensated K-type thermocouple digitizer. It outputs temperature in 0.25°C increments via SPI.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `src/main.cpp` | Entry point, setup(), loop(), WebSocket handling |
| `include/temperature.h` | `isValidTemp()`, `RollingAverage<N>` template |
| `include/telemetry.h` | `buildTelemetryJson()` — builds the WebSocket JSON frame |

---

## Data Flow

```
┌─────────────┐
│ MAX6675     │  readCelsius() ────► rawBt, rawEt (float)
│ (BT & ET)   │
└─────────────┘
       │
       ▼
┌─────────────┐
│ isValidTemp │  Returns false if:
│   (temp.h)  │    • t < 0.5°C  (VCC/GND missing)
│             │    • t > 1000°C  (thermocouple open/disconnected)
└─────────────┘
       │
       ▼
┌─────────────┐
│ Rolling     │  5-sample rolling average
│ Average<5>  │  Smooths sensor jitter (4 Hz sampling)
│  (temp.h)   │
└─────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ Critical section (dataMux)          │
│ g_bt, g_et (volatile globals)       │
└─────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ WebSocket /ws                       │
│ Artisan connects and receives:       │
│ {"id":N,"data":{"BT":185.5,"ET":210}}│
└─────────────────────────────────────┘
```

---

## Pin Definitions (`main.cpp:9-11`)

```cpp
static constexpr int BT_DO  = 19, BT_CS  = 23, BT_CLK = 5;   // Bean Temp
static constexpr int ET_DO  = 33, ET_CS  = 25, ET_CLK = 26;   // Exhaust Temp
```

These are defined as compile-time constants (`static constexpr`) for flash efficiency.

---

## Timing (`main.cpp:13-18`)

```cpp
static constexpr unsigned long READ_INTERVAL_MS      = 250UL;   // 4 Hz sensor reads
static constexpr unsigned long WS_CLEANUP_INTERVAL_MS = 1000UL;
static constexpr unsigned long SERIAL_LOG_INTERVAL_MS = 1000UL; // 1-second serial log
static constexpr unsigned long HEAP_LOG_INTERVAL      = 30000UL;
static constexpr uint32_t      HEAP_WARN_BYTES        = 10000UL;
```

- **250ms interval** = 4 readings per second per sensor
- **1-second serial log** = human-readable output for debugging
- **30-second heap monitor** = catches memory exhaustion early

---

## Temperature Validation (`include/temperature.h`)

```cpp
static constexpr float TEMP_MIN_VALID = 0.5f;
static constexpr float TEMP_MAX_VALID = 1000.0f;

inline bool isValidTemp(float t) {
    return t >= TEMP_MIN_VALID && t <= TEMP_MAX_VALID;
}
```

The MAX6675 returns specific sentinel values for fault conditions:

| Value | Meaning |
|-------|---------|
| `0.0°C` | VCC or GND missing (no power) |
| `1023.75°C` | Thermocouple open/disconnected |

These are outside the valid range and are rejected.

---

## Rolling Average (`include/temperature.h:15-40`)

```cpp
template<uint8_t N>
class RollingAverage {
    void push(float v) {
        buf[idx] = v;
        idx = (idx + 1) % N;
        if (count < N) ++count;
    }
    float value() const { /* sum / count */ }
    // ...
};
```

A fixed-capacity circular buffer — **no heap allocation**. With `N=5` and 4 Hz sampling, the window covers ~1.25 seconds of data.

---

## WebSocket Server (`main.cpp:38-68`)

The ESP32 runs an **AsyncWebServer** on port 80 with a WebSocket handler at `/ws`:

```cpp
static AsyncWebSocket ws("/ws");

static void onWsEvent(AsyncWebSocket* /*srv*/, AsyncWebSocketClient* client,
                      AwsEventType type, void* arg, uint8_t* data, size_t len) {
    if (type == WS_EVT_CONNECT) {
        Serial.printf("[WS] Client #%u connected from %s\n",
                      client->id(), client->remoteIP().toString().c_str());
    } else if (type == WS_EVT_DISCONNECT) {
        Serial.printf("[WS] Client #%u disconnected\n", client->id());
    } else if (type == WS_EVT_DATA) {
        // Artisan sends a text message → respond with latest BT/ET
        float bt, et;
        taskENTER_CRITICAL(&dataMux);
        bt = g_bt; et = g_et;
        taskEXIT_CRITICAL(&dataMux);

        char buf[TELEMETRY_BUF_LEN];
        size_t n = buildTelemetryJson(buf, sizeof(buf), reqId, bt, et);
        if (n > 0) client->text(buf, n);
    }
}
```

**Important:** Artisan initiates the connection. The ESP32 does NOT connect to Artisan — it hosts the WebSocket server and waits for Artisan to connect.

---

## WebSocket Protocol (`include/telemetry.h`)

The JSON frame format sent to Artisan:

```json
{"id":1,"data":{"BT":185.50,"ET":210.25}}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Incrementing message counter |
| `data.BT` | float | Bean Temperature in Celsius |
| `data.ET` | float | Exhaust Temperature in Celsius |

Frame size: ~75 bytes maximum, well within WebSocket MTU.

---

## WiFi Access Point Mode (`main.cpp:83-87`)

```cpp
WiFi.mode(WIFI_AP);
WiFi.softAP(AP_SSID);  // AP_SSID = "RoastIQ"
Serial.printf("[WiFi] AP \"%s\" started — IP: %s\n",
              AP_SSID, WiFi.softAPIP().toString().c_str());
```

The ESP32 creates an open access point named **RoastIQ**. No password is required. This means:

- No WiFi credentials to provision
- Works anywhere — no router needed
- Artisan (or a phone/laptop) connects directly to the ESP32

---

## The Loop (`main.cpp:98-151`)

The main loop runs at full speed but uses time-gated checks:

```cpp
void loop() {
    unsigned long now = millis();

    // 1. Read sensors at 4 Hz (every 250ms)
    if (now - lastReadMs >= READ_INTERVAL_MS) {
        float rawBt = sensorBT.readCelsius();
        float rawEt = sensorET.readCelsius();
        if (btOk) btAvg.push(rawBt);
        if (etOk) etAvg.push(rawEt);
        // Store smoothed values in critical section
        taskENTER_CRITICAL(&dataMux);
        g_bt = btAvg.value(); g_et = etAvg.value();
        taskEXIT_CRITICAL(&dataMux);
    }

    // 2. Log to serial at 1 Hz
    if (now - lastSerialLogMs >= SERIAL_LOG_INTERVAL_MS) {
        Serial.printf("[TEMP] BT: %.1f°C  ET: %.1f°C\n", bt, et);
    }

    // 3. Clean up disconnected WebSocket clients
    if (ws.count() > 0 && (now - lastWsCleanMs >= WS_CLEANUP_INTERVAL_MS)) {
        ws.cleanupClients();
    }

    // 4. Monitor heap every 30s
    if (now - lastHeapMs >= HEAP_LOG_INTERVAL) {
        uint32_t freeHeap = ESP.getFreeHeap();
        if (freeHeap < HEAP_WARN_BYTES) Serial.println("[WARN] Heap critically low");
    }
}
```

No `delay()` is used — the loop spins freely, spending most of its time in `delay()`-like checks. This maximizes responsiveness.

---

## Building and Flashing

### Development Build (verbose logging, erases NVS)

```bash
pio run -e dev --target upload --upload-port /dev/ttyUSB0
pio device monitor --port /dev/ttyUSB0 --baud 115200
```

### Production Build

```bash
pio run -e esp32doit-devkit-v1 --target upload --upload-port /dev/ttyUSB0
```

### Running Tests (no hardware needed)

```bash
pio run -e native  # runs temperature + telemetry unit tests
```

---

## Troubleshooting

### ESP32 won't flash
Hold the **BOOT** button on the ESP32 while pressing **EN** (reset). Release BOOT after the upload starts.

### Permission denied on /dev/ttyUSB0
```bash
sudo usermod -aG dialout $USER
# Log out and back in
```

### Artisan shows "0.0" for BT and ET
- Check thermocouple connections — ensure they're firmly seated in the MAX6675 screw terminals
- Try a known-good K-type thermocouple
- Check serial monitor for `[TEMP]` readings to confirm ESP32 is reading valid values

### WebSocket connection refused
- Confirm the IP address in Artisan's WebSocket URL matches the serial output
- Ensure no firewall is blocking port 80
- Try connecting a phone/laptop to the RoastIQ AP first to verify the server is running
