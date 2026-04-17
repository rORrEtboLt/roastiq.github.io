# RoastIQ vs. Artisan Scope

RoastIQ and Artisan Scope are **complementary tools**, not competitors. Understanding what each does best helps you use them together effectively.

---

## Artisan Scope — The Industry Standard

**Artisan** is the de facto standard for coffee roasting software. It is a mature, feature-rich desktop application used by roasters worldwide.

### What Artisan Does Best

| Feature | Description |
|---------|-------------|
| **Roast profiling** | Precise curve control with configurable axes, rate-of-rise (RoR) display |
| **Phase alarming** |触发事件 (CHARGE, DRY END, 1C START, etc.) with customizable alarms |
| **Data export** | Export roast profiles as CSV, JSON, PDF reports |
| **Blend tool** | Calculate blend ratios and track blend roasts |
| **Modbus/ITC support** | Connect to commercial roasting equipment directly |
| **Community sharing** | Large library of shared roast profiles and recipes |
| **Chard / Artisan Zoom** | Companion apps for live monitoring |

### Artisan's Limitations

- Desktop-only (no web access)
- Single-device monitoring (one machine per Artisan instance)
- No team collaboration features
- No cloud sync — profiles stored locally
- The web-based "Artisan Zoom" requires additional setup

---

## RoastIQ — Cloud-Connected Roaster Management

**RoastIQ** is a modern take on roasting telemetry: an ESP32-based hardware system with optional cloud connectivity.

### What RoastIQ Does Best

| Feature | Description |
|---------|-------------|
| **Affordable hardware** | Uses off-the-shelf ESP32 + MAX6675 (under $15 of electronics) |
| **Simple setup** | Flash the ESP32, connect thermocouples, start roasting |
| **Web dashboard** | Monitor roasts from any browser — phone, tablet, desktop |
| **Multi-roaster** | Manage multiple roasters from a single dashboard |
| **Team access** | Share roast data with your team in real-time |
| **Cloud history** | All roasts automatically saved and accessible |
| **Open source** | Full firmware, server, and web app — audit, modify, contribute |

### RoastIQ's Limitations

- No native roast profiling tools (uses Artisan for curve editing)
- No direct Modbus/ITC support ( thermocouples only)
- No blend tool or recipe management
- Requires internet for cloud features (standalone mode = Artisan only)

---

## How They Work Together

RoastIQ's ESP32 streams data at 4 Hz. Both Artisan and the RoastIQ server can consume this data simultaneously:

```
┌──────────────┐      4 Hz WebSocket       ┌──────────────────┐
│   ESP32      │ ─────────────────────────►│  Artisan Scope    │
│  (RoastIQ)   │                           │  (desktop)        │
└──────────────┘                           └──────────────────┘
       │
       │ 1 Hz REST
       ▼
┌──────────────┐      Socket.io / HTTP       ┌──────────────────┐
│  RoastIQ    │ ─────────────────────────►│  Web Dashboard   │
│  Server     │                           │  (any browser)   │
└──────────────┘                           └──────────────────┘
```

**In practice:**
1. Connect Artisan directly to the ESP32 via WebSocket for your primary roast profiling and alarming
2. Simultaneously, the ESP32 (or Artisan) sends data to the RoastIQ server
3. View your roast on the RoastIQ web dashboard from anywhere
4. Share a live link with your team during a roast

---

## Comparison Table

| Feature | Artisan Scope | RoastIQ |
|---------|-------------|---------|
| **Platform** | Desktop (macOS, Windows, Linux) | ESP32 + Web (any browser) |
| **Cost** | Free (donations welcome) | Hardware ~$15 + optional server |
| **Temperature source** | Modbus, ITC, HTTP, WebSocket, more | MAX6675 (K-type thermocouple) |
| **Real-time chart** | Yes | Yes |
| **Rate-of-Rise (RoR)** | Yes | Yes |
| **Phase alarms** | Yes | Via server integration |
| **Data export** | CSV, JSON, PDF | CSV (via server API) |
| **Multi-roaster** | No | Yes |
| **Team collaboration** | No | Yes (web dashboard) |
| **Cloud sync** | No (local only) | Yes |
| **Mobile access** | Via Artisan Zoom | Yes (web) |
| **Open source** | Partial (Python base) | Full (MIT) |
| **Internet required** | No | For cloud features |

---

## Recommended Setup

### For Home Roasters
Use **Artisan alone** with the ESP32 via direct WebSocket. Free, simple, and gives you everything you need for roast profiling.

### For Small Coffee Shops
Use **Artisan** for roasting (profile, alarms, export). Add **RoastIQ** if you want to monitor roasts from your phone or share data with staff.

### For Specialty Coffee Operations
Run **both simultaneously**:
- Artisan as the primary roasting interface (profiling, alarms, data export)
- RoastIQ web dashboard for real-time monitoring from multiple devices
- RoastIQ server provides roast history and team access

### For Multiple Roasters or Franchises
**RoastIQ's multi-roaster management** shines here:
- One web dashboard monitors all roasters
- Role-based access (roaster, manager, owner)
- Historical comparison across machines
- No need for each workstation to run Artisan

---

## Summary

- **Artisan** = the gold standard for roast profiling and control
- **RoastIQ** = modern cloud-connected telemetry for team collaboration and remote monitoring
- **Together** = best of both worlds, sharing the same ESP32 data source
