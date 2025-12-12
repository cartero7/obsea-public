# OBSEA3 — Submarine Power & Instrument Control

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

> **Designed by**: Carla Artero Delgado (UPC) — Catalunya  
> **License**: See LICENSE file

## ⚡ Quick Start (Installation from Scratch)

**Requirements**: Ubuntu 24.04 LTS (Noble Numbat) - x86_64 or ARM64

```bash
# Clone repository
git clone https://github.com/cartero7/obsea.git
cd obsea

# Install everything
sudo bash bin/00_install_all.sh
```

**That's it!** The automated installer will:
- ✅ Guide you through configuration (hostname, ports, etc.)
- ✅ Request sudo only when needed
- ✅ Install all dependencies automatically
- ✅ Build the ROS2 workspace
- ✅ Create default user (username: `test`, password: `test123`, role: viewer)
- ✅ Verify installation
- ✅ Optionally start the system

### Post-Installation (Optional)

```bash
# Create default user and configure backups
sudo bash bin/99_post_install.sh

# Start all services
sudo /opt/obsea/bin/50_run_all.sh
```

**Optional:** Check system requirements first:
```bash
bash bin/check_requirements.sh
```

📖 **Quick installation guide:** [docs/INSTALL_QUICKSTART.md](docs/INSTALL_QUICKSTART.md)  
📖 **Full installation guide:** [docs/INSTALL.md](docs/INSTALL.md)  
📖 **User management & backups:** [docs/USUARIOS_Y_DATOS.md](docs/USUARIOS_Y_DATOS.md)

---

## Documentation

Choose your language:
- 🇬🇧 English: [docs/README.en.md](docs/README.en.md)
- 🇪🇸 Español: [docs/README.es.md](docs/README.es.md)

**Key docs:**
- [docs/INSTALL.md](docs/INSTALL.md) — Quick start installation guide
- [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md) — Full setup, config, troubleshooting
- [CHANGELOG.md](CHANGELOG.md) — **Version history and release notes**
- [docs/WORKFLOW.md](docs/WORKFLOW.md) — **Development workflow** (edit → build → test → commit)
- [bin/README.md](bin/README.md) — Script reference and workflow
- [docs/AUTHORS.md](docs/AUTHORS.md) — Project contributors

---

## Manual Installation (5 steps)

```bash
# 1) Generate config locally (creates bin/.venv, bin/secrets)
bash bin/00_interactive_setup.sh

# 2) Setup host (creates /opt/obsea dirs, copies code, sets permissions)
sudo bash bin/02_host_setup.sh

# 3) Install Python dependencies
sudo bash bin/12_install_python_deps.sh

# 4) Build ROS2 workspace
bash bin/30_build_ros2_ws.sh

# 5) Start system
bash bin/50_run_all.sh
```
Health: `curl http://localhost:8101/health`

**Default credentials:**
- Username: `test`
- Password: `test123`
- Role: `viewer` (read-only, cannot control ports)

Create admin users with: `python3 bin/more/obsea_user_admin.py`

## Layout
- `ros2_ws/src/` — ROS2 packages (API gateway, core nodes, hardware interfaces, interfaces, web proxy)
- `bin/` — deployment/admin scripts; legacy in `bin/archived/`
- `webtest/` — Vite/React frontend (ignored by default; remove from `.gitignore` to track)
- `.github/ARCHITECTURE.md` — architecture overview

## Common tasks
- Quick validation: `bash bin/quick_test.sh`
- Full validation: `bash bin/comprehensive_test.sh`
- User management (sudo): `bash bin/03_create_users.sh`
- Logs: `/opt/obsea/log/current/`
- Secrets: template `bin/secrets.example`; real secrets in `/opt/obsea/bin/secrets`

## Service Management (systemd)

After installation, OBSEA3 runs as persistent systemd services:

```bash
# Ver si están habilitados/corriendo
sudo systemctl list-unit-files 'obsea3-*'    # Ver enabled/disabled
sudo systemctl is-active obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Ver estado detallado
sudo systemctl status 'obsea3-*'

# Habilitar y arrancar todos los servicios
sudo systemctl enable obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix
sudo systemctl start obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Reiniciar todos los servicios
sudo systemctl restart obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Parar/arrancar servicios individuales
sudo systemctl stop obsea3-hw-supervisor
sudo systemctl start obsea3-api-gateway

# Ver logs en tiempo real
journalctl -u obsea3-hw-supervisor -f
tail -f /opt/obsea/log/current/hw_supervisor.log
```

**Services:**
- `obsea3-hw-supervisor` - Hardware control node (ROS2)
- `obsea3-api-gateway` - FastAPI + ROS2 bridge
- `obsea3-web` - Vite dev server (React frontend)

All services auto-start on boot and auto-restart on failure.

## Notes
- Build artifacts (`ros2_ws/build`, `ros2_ws/install`) and `webtest/node_modules` are ignored.
- To version the frontend, remove `webtest/` from `.gitignore`.

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
