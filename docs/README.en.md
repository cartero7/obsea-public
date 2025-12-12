# OBSEA3 — Submarine Power & Instrument Control

ROS2-based system with FastAPI backend and optional Vite/React frontend.

```
╔════════════════════════════════════════════════════════════════╗
║            🌊 OBSEA3 System Architecture 🌊                    ║
╚════════════════════════════════════════════════════════════════╝

   ┌──────────────────────────────────────────────────────────┐
   │  🌐 Web Interface (React + Vite)                         │
   │  ├─ Dashboard & Real-time Telemetry                      │
   │  ├─ Port Control (12V/48V switching)                     │
   │  └─ User Management & Config Editor                      │
   └──────────────────────┬───────────────────────────────────┘
                          │ HTTP/REST + WebSocket
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  🚀 API Gateway (FastAPI)                                │
   │  ├─ JWT Authentication & Port Leases                     │
   │  ├─ Audit Logging & Safety Checks                        │
   │  └─ ROS2 Bridge (Topics ↔ REST/WS)                       │
   └──────────────────────┬───────────────────────────────────┘
                          │ ROS2 Services & Topics
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  🤖 Hardware Supervisor (ROS2 Jazzy)                     │
   │  ├─ I2C Sensors (LTC2945, MCP23008, ADS1115)             │
   │  ├─ SNMP Network Monitoring                              │
   │  ├─ Safety Manager (Voltage/Current Limits)              │
   │  └─ State Machine & Alarm System                         │
   └──────────────────────┬───────────────────────────────────┘
                          │ I2C Bus + GPIO + SNMP
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  ⚡ Hardware Backplane (Custom PCB)                       │
   │  └─ Power Rails: 48V / 12V / 5V with monitoring          │
   └──────────────────────┬───────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
       ┌──────▼──────┐         ┌──────▼──────┐
       │ 🔌 Port 1-4 │   ...   │ 🔌 Port 5-8 │
       │  (12V/48V)  │         │  (12V/48V)  │
       └──────┬──────┘         └──────┬──────┘
              │                       │
              └───────────┬───────────┘
                          ↓
                 ┌────────────────────┐
                 │  🪼 Underwater Lab  │
                 │  ├─ CTD Sensors    │
                 │  ├─ IP Cameras     │
                 │  ├─ Hydrophones    │
                 │  └─ Instruments    │
                 └────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │  📊 Monitoring & Observability                           │
   │  ├─ Zabbix Bridge → 📈 Metrics Server                    │
   │  ├─ SQLite Audit Logs (Actions + Alarms)                 │
   │  └─ Config Versioning (YAML Archives)                    │
   └──────────────────────────────────────────────────────────┘
```

> **Designed by**: Carla Artero Delgado (UPC) — Catalunya  
> **License**: See LICENSE file

## Documentation
- `INSTALL.md` — Quick start installation
- `DEPLOYMENT_GUIDE.md` — Full setup, config, troubleshooting
- `docs/WORKFLOW.md` — **Development workflow** (edit → build → test → commit)
- `bin/README.md` — Script reference
- `.github/ARCHITECTURE.md` — Architecture cheat sheet

## Installation (5 steps)

**Requirements**: Ubuntu 24.04 LTS (Noble Numbat) - x86_64 or ARM64

```bash
# 1) Generate config (creates .venv, secrets template)
bash bin/00_interactive_setup.sh

# 2) Setup host (creates dirs, copies code, sets permissions)
sudo bash bin/02_host_setup.sh

# 3) Install Python dependencies
sudo bash bin/12_install_python_deps.sh

# 4) Build ROS2 workspace
bash bin/30_build_ros2_ws.sh

# 5) Start system
bash bin/50_run_all.sh
```
Health: `curl http://localhost:8101/health`

**Default login credentials:**
- Username: `test`
- Password: `test123`  
- Role: `viewer` (read-only access)

Create admin users: `python3 bin/more/obsea_user_admin.py`

## Layout
- `ros2_ws/src/` — ROS2 packages (API gateway, core nodes, hardware interfaces, interfaces, web proxy)
- `bin/` — deployment/admin scripts; legacy in `bin/archived/`
- `webtest/` — Vite/React frontend (ignored by default; remove from `.gitignore` to track)
- `.github/ai-agent-instructions.md` — condensed architecture overview

## Common tasks
- Quick validation: `bash bin/quick_test.sh`
- Full validation: `bash bin/comprehensive_test.sh`
- User management (sudo): `bash bin/03_create_users.sh`
- Logs (runtime host): `/opt/obsea/log/current/`
- Secrets: template `bin/secrets.example`; real secrets in `/opt/obsea/bin/secrets`

## Notes
- Build artifacts (`ros2_ws/build`, `ros2_ws/install`) and `webtest/node_modules` are ignored.
- If you need the frontend in git, remove `webtest/` from `.gitignore`.

---

## Credits & License

**System Design & Development**:  
🧑‍💻 **Carla Artero Delgado**  
📍 Universitat Politècnica de Catalunya (UPC)  
🌍 Catalunya

**Project**: OBSEA3 Submarine Observatory Control System  
**Technologies**: ROS2 Jazzy, FastAPI, React, Python, I2C, SNMP

**Repository**: [github.com/cartero7/obsea](https://github.com/cartero7/obsea)

---

Made with 🪼 for underwater science
