# OBSEA3 Public Documentation

This folder contains public-facing documentation in English. It is designed to be synced into a public repository (no private code required).

## System at a glance
OBSEA3 — Submarine Power & Instrument Control  
ROS2-based system with FastAPI backend and React web frontend.

```
┌────────────────────────────────────────────────────────────────┐
│              OBSEA3 SYSTEM ARCHITECTURE                        │
└────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Web Interface (React + Vite)                                │
│  • Dashboard & Real-time Telemetry                           │
│  • Port Control (12V/48V switching)                          │
│  • User Management & Config Editor                           │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTP/REST + WebSocket
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  API Gateway (FastAPI)                                       │
│  • JWT Authentication & Port Leases                          │
│  • Audit Logging & Safety Checks                             │
│  • ROS2 Bridge (Topics ↔ REST/WS)                            │
└────────────────────────┬─────────────────────────────────────┘
                         │ ROS2 Services & Topics
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  Hardware Supervisor (ROS2 Jazzy)                            │
│  • I2C Sensors (LTC2945, MCP23008, ADS1115)                  │
│  • SNMP Network Monitoring                                   │
│  • Safety Manager (Voltage/Current Limits)                   │
│  • State Machine & Alarm System                              │
└────────────────────────┬─────────────────────────────────────┘
                         │ I2C Bus + GPIO + SNMP
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  Hardware Backplane (Custom PCB)                             │
│  • Power Rails: 48V / 12V / 5V with monitoring               │
└────────────────────────┬─────────────────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
      ┌─────▼──────┐            ┌─────▼──────┐
      │ Port 1-4   │    ...     │ Port 5-8   │
      │ (12V/48V)  │            │ (12V/48V)  │
      └─────┬──────┘            └─────┬──────┘
            │                         │
            └────────────┬────────────┘
                         ↓
                ┌─────────────────┐
                │ Underwater Lab  │
                │ • CTD Sensors   │
                │ • IP Cameras    │
                │ • Hydrophones   │
                │ • Instruments   │
                └─────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Monitoring & Observability                                  │
│  • Zabbix Bridge → Metrics Server                            │
│  • SQLite Audit Logs (Actions + Alarms)                      │
│  • Config Versioning (YAML Archives)                         │
└──────────────────────────────────────────────────────────────┘
```

## Start here
- [API.md](API.md) - REST API login, telemetry, leases, and safe port control.
- [ROS2_CLI.md](ROS2_CLI.md) - Using the ROS2 CLI with JWTs or the system token.
- [ROS2_CLI_examples.md](ROS2_CLI_examples.md) - Ready-to-run scripts for operator, maintenance, and one-shot flows.
- [USAGE.md](USAGE.md) - Quick index (legacy entry point).

## Notes
- These docs are intentionally minimal and avoid private deployment details.
- Internal and operational docs live in the private repository under `docs/`.

## Contact
For questions or support: carola.artero@upc.edu

## Credits & License

**System Design & Development**:  
🧑‍💻 **Carla Artero Delgado**  
📍 Universitat Politècnica de Catalunya (UPC)  
🌍 Catalunya  
📧 carola.artero@upc.edu

**Project**: OBSEA3 Submarine Observatory Control System  
**Technologies**: ROS2 Jazzy, FastAPI, React, Python, I2C, SNMP
