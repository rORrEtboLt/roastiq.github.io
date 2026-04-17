# System Architecture

This document describes the full RoastIQ system architecture — from ESP32 sensor reading to web dashboard display.

---

## Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RoastIQ System                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐                                                           │
│   │   ESP32      │◄── USB/Serial ──── Flashing (PlatformIO)                  │
│   │  (Firmware)  │                                                           │
│   │              │                                                           │
│   │  MAX6675 BT  │◄── SPI ──────── K-type thermocouple (roast chamber)      │
│   │  MAX6675 ET  │◄── SPI ──────── K-type thermocouple (exhaust duct)        │
│   │              │                                                           │
│   │  WiFi AP     │◄── WiFi ─────── Artisan Scope (WebSocket :80/ws)        │
│   │  WebSocket   │◄── WiFi ─────── RoastIQ Server (REST POST :3001)         │
│   └──────┬───────┘                                                           │
│          │                                                                    │
│          │ WebSocket / REST                                                  │
│          ▼                                                                    │
│   ┌──────────────┐         ┌─────────────────┐         ┌─────────────────┐ │
│   │  Artisan     │         │  RoastIQ Server  │         │   Web Frontend   │ │
│   │  Scope       │         │  (Node.js)        │         │   (React)        │ │
│   │              │◄──────►│                  │◄───────►│                  │ │
│   │  • Live chart│  Socket │  • Express API   │  REST   │  • Roast curves  │ │
│   │  • Phase marks│ .io    │  • Prisma ORM    │  HTTP   │  • Event marking │ │
│   │  • Data export│        │  • PostgreSQL     │         │  • History       │ │
│   └──────────────┘         └────────┬────────┘         └─────────────────┘ │
│                                      │                                    │
│                                      ▼                                    │
│                             ┌─────────────────┐                           │
│                             │   PostgreSQL    │                           │
│                             │                  │                           │
│                             │  • Organizations │                           │
│                             │  • Users        │                           │
│                             │  • Devices      │                           │
│                             │  • Roasts       │                           │
│                             │  • Samples       │                           │
│                             │  • Events        │                           │
│                             └─────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## ESP32 Firmware (`src/`)

The firmware is built with **PlatformIO** targeting the **ESP32 Arduino framework**.

| File | Purpose |
|------|---------|
| `src/main.cpp` | Setup, loop, WebSocket server, sensor reading |
| `include/temperature.h` | `isValidTemp()`, `RollingAverage<N>` |
| `include/telemetry.h` | `buildTelemetryJson()` |

**Key characteristics:**
- Runs an **AsyncWebServer** with WebSocket support on port 80
- **4 Hz** sensor sampling (250ms interval)
- 5-sample rolling average smoothing
- Critical sections protect shared float variables (`g_bt`, `g_et`)
- No heap allocations in the hot path

---

## RoastIQ Server (`server/`)

A **Node.js + Express** application with **Socket.io** for real-time communication.

### Stack

| Layer | Technology |
|-------|------------|
| Runtime | Node.js |
| HTTP Framework | Express |
| Real-time | Socket.io |
| Database ORM | Prisma |
| Database | PostgreSQL |
| Auth | JWT (jsonwebtoken) |
| Email | Resend / Nodemailer (SMTP) |

### API Routes

#### Public (no auth required)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/login` | Login, returns JWT |
| POST | `/api/auth/register` | Register with invite code |
| POST | `/api/auth/refresh` | Refresh access token |
| GET | `/api/health` | Health check |
| POST | `/api/device/telemetry` | ESP32 pushes BT/ET data |
| GET | `/api/artisan/current` | Artisan HTTP polling (when server-connected) |
| POST | `/api/artisan/event` | Artisan sends phase events |
| POST | `/api/device/auth` | Device authentication |

#### Authenticated (requires JWT)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/status` | ESP32 connection + active roast status |
| GET/POST/DELETE | `/api/roasts/*` | Roast CRUD |
| GET/POST | `/api/events/*` | Roast event recording |
| GET | `/api/org/users` | Org user management |
| GET/POST/DELETE | `/api/org/invites` | Invite code management |
| GET/POST/DELETE | `/api/org/devices` | Device registration |

#### Super Admin only
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/orgs` | List all organizations |
| POST | `/api/orgs` | Create organization |
| DELETE | `/api/orgs/:id` | Delete organization |

### Socket.io Events

The server broadcasts real-time events to connected clients (web dashboard).

#### Server → Client
| Event | Payload | When |
|-------|---------|------|
| `status` | `{esp32Connected, esp32Target, activeRoastId}` | On connect |
| `temperature` | `{bt, et, rorBt, elapsedS}` | Every 1s (ESP32 connected) |
| `sample` | `{roastId, elapsedS, bt, et, rorBt}` | During active roast |
| `event` | `{roastId, elapsedS, label, bt}` | When event is marked |
| `roast:started` | `{roastId}` | On CHARGE |
| `roast:ended` | `{roastId}` | On DROP |
| `esp32:connected` | `{ip, port, orgId}` | ESP32 connects |
| `esp32:disconnected` | — | ESP32 disconnects |

#### Client → Server
| Event | Description |
|-------|-------------|
| `join:org` | SUPER_ADMIN switches org room for live view |

---

## Web Frontend (`web/`)

A **React + Vite** application.

### Stack

| Layer | Technology |
|-------|------------|
| Framework | React 18 + Vite |
| Routing | React Router |
| Real-time | Socket.io client |
| Charts | ( roast profiling chart ) |
| State | React hooks (useState, useContext) |

### Key Components

| Component | Purpose |
|-----------|---------|
| `RoastChart.tsx` | Real-time roast curve with BT/ET/ROR |
| `TemperatureDisplay.tsx` | Current BT/ET readout |
| `RoastTimer.tsx` | Elapsed time since CHARGE |
| `EventBar.tsx` | Phase event markers (CHARGE, 1C, DROP, etc.) |
| `RoastHistory.tsx` | Past roasts list |
| `AdminPanel.tsx` | Device management, user management |
| `LoginPage.tsx` / `RegisterPage.tsx` | Auth |

---

## Database Schema (Prisma)

The database uses **PostgreSQL** with **Prisma ORM**.

### Core Models

```
Organization
  ├── id, name, createdAt
  ├── paymentExpiresAt
  └── users[], devices[], roasts[]

User
  ├── id, email, password, name, role
  └── orgId → Organization

Device (ESP32)
  ├── id, name, macAddress
  ├── ip, port
  └── orgId → Organization

Roast
  ├── id, profile[], roastEvents[]
  ├── startedAt, endedAt
  └── orgId → Organization
  └── samples[]

Sample
  ├── t (elapsed seconds), bt, et, rorBt
  └── roastId → Roast

RoastEvent
  ├── elapsedS, label, bt, sourceDevice
  └── roastId → Roast
```

---

## Data Flow: Temperature Reading to Web Dashboard

```
1. ESP32 reads MAX6675 sensors at 4 Hz
        │
2. isValidTemp() validates readings
        │
3. RollingAverage<5> smooths values
        │
4. Values stored in g_bt, g_et (critical section)
        │
5. WebSocket: Artisan connects and receives:
   {"id":N,"data":{"BT":185.5,"ET":210.3}}
        │
   OR (server-connected mode)
        │
5. REST POST /api/device/telemetry
   Body: {bt, et, elapsedS, deviceToken}
        │
6. roastManager.ingestSample() in server
   - Stores in rolling RoR window
   - Emits Socket.io 'temperature' to org room
   - Saves to PostgreSQL sample buffer
        │
7. React web client receives via Socket.io
        │
8. useSocket.ts hook updates React state
        │
9. RoastChart.tsx re-renders with new data point
```

---

## Rate Summary

| Path | Rate | Notes |
|------|------|-------|
| ESP32 → MAX6675 (SPI) | 4 Hz | 250ms sensor reads |
| ESP32 → WebSocket (Artisan) | 4 Hz | JSON frames |
| ESP32 → Server (REST) | 1 Hz | Telemetry POST |
| Server → Web Client (Socket.io) | 1 Hz | Temperature broadcast |
| Server → Web Client (Socket.io) | On-demand | Events, roast start/end |
| Web Client → Server (REST) | On-demand | Event marking, roast CRUD |
