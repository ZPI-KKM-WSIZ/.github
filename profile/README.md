# ZPI Air Quality

A distributed air quality monitoring system that collects particulate matter and environmental
data from physical sensor stations and presents it on an interactive public map.

The system is composed of five repositories working in concert: embedded firmware running on
Arduino hardware, a shared contracts library, a Cassandra database layer, a stateless FastAPI
backend cluster, and a Node.js/Express frontend.

---

## System Overview

```
  [Sensors] ───┬─── [Frontend]
               ▼
        [Public Domain]
               │
               │  [Cloudflare Tunnel]
               │
        ┌──────┴──────┐
        ▼             ▼
  Backend Node   Backend Node ← stateless, replicable
        └──────┬──────┘
               │ [Tailscale VPN]
               ▼
      [Cassandra Cluster]

```

**Data flow:**

1. Each physical sensor station (Arduino UNO R4 WiFi) periodically POSTs a
   `SensorReadingDTO` payload to the public backend API.
2. A stateless backend node validates and persists the reading to the Cassandra cluster
   over a Tailscale VPN mesh.
3. The frontend aggregates the latest readings and locations from the backend and renders
   them as colour-coded markers on a Leaflet map.
4. The browser periodically auto-refreshes to reflect new data.

---

## Repositories

| Repository                                                         | Language   | Description                                                       |
|--------------------------------------------------------------------|------------|-------------------------------------------------------------------|
| [contracts](https://github.com/ZPI-KKM-WSIZ/contracts)             | Python     | Shared Pydantic models and DTOs used by all Python services       |
| [database](https://github.com/ZPI-KKM-WSIZ/database)               | Python     | Cassandra connection management, migrations, and repository layer |
| [backend](https://github.com/ZPI-KKM-WSIZ/backend)                 | Python     | Stateless FastAPI backend — ingestion, API, and Cassandra service |
| [frontend](https://github.com/ZPI-KKM-WSIZ/frontend)               | JavaScript | Node.js/Express server with interactive Leaflet map               |
| [sensor_firmware](https://github.com/ZPI-KKM-WSIZ/sensor_firmware) | C++        | Arduino UNO R4 WiFi firmware for PMS7003 + BME680 + DHT11         |

---

## Components

### Contracts

A shared Python package providing Pydantic models (DTOs) imported by the `database` and
`backend` services. This single source of truth ensures consistent data shapes across the
entire system.

See the [contracts repository](https://github.com/ZPI-KKM-WSIZ/contracts) for full details.

---

### Database

A shared Cassandra database layer providing connection management, ordered CQL schema
migrations, typed repository abstractions, and automated Tailscale-aware node deployment.
The Cassandra cluster spans multiple nodes connected via a Tailscale VPN mesh; on the first
node, migrations are applied automatically at bootstrap time.

See the [database repository](https://github.com/ZPI-KKM-WSIZ/database) for setup,
configuration, and deployment documentation.

---

### Backend

A stateless FastAPI backend that ingests `SensorReadingDTO` payloads from sensor stations,
persists them to the Cassandra cluster, and serves a REST API consumed by the frontend.
Multiple identical nodes can run simultaneously — Cloudflare Tunnel handles public HTTPS
exposure and load balancing. Each node auto-generates a unique `SERVER_ID` at startup for
observability.

See the [backend repository](https://github.com/ZPI-KKM-WSIZ/backend) for configuration,
deployment, and API reference documentation.

---

### Frontend

A lightweight Node.js/Express server that aggregates sensor readings and locations from
the backend and renders them as colour-coded circle markers on an interactive Leaflet map.
It supports station search, address geocoding, GPS centering, per-sensor detail panels
(PM1, PM2.5, PM10, temperature, humidity, pressure), and a persistent light/dark theme.
All other API calls are proxied transparently to the backend.

See the [frontend repository](https://github.com/ZPI-KKM-WSIZ/frontend) for setup and
configuration documentation.

---

### Sensor Firmware

PlatformIO firmware for an **Arduino UNO R4 WiFi** station, reading from a PMS7003
(PM1/PM2.5/PM10 via UART), BME680 (temperature, humidity, pressure via I²C), and DHT11
(temperature and humidity). The station operates in STA mode with AP fallback, exposes a
local web panel (Home / Settings / API), and persists configuration to EEPROM.

See the [sensor_firmware repository](https://github.com/ZPI-KKM-WSIZ/sensor_firmware) for
build instructions, wiring diagrams, and debug documentation.

---

## Infrastructure

The system uses two networking technologies to connect components securely:

- **Tailscale VPN** — backend nodes and Cassandra nodes form a private mesh network.
  Cassandra nodes are tagged and discovered automatically by the deployer and backend
  services at startup. No public ports are exposed for the database.
- **Cloudflare Tunnel** — the backend and frontend are exposed to the public internet
  via Cloudflare Zero Trust tunnels, eliminating the need for open inbound ports or
  static IP addresses.
