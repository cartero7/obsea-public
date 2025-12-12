# OBSEA3 Deployment Guide

Complete installation guide for OBSEA3 submarine control system.

> 📖 **After installation**, see [docs/WORKFLOW.md](docs/WORKFLOW.md) for development workflow (edit → build → test → commit)

---

## 🚀 Quick Installation

**Automated One-Command Install (Recommended):**

```bash
bash bin/00_install_all.sh
```

This interactive script will:
- ✅ Generate configuration with wizard
- ✅ Create system directories and users
- ✅ Install all dependencies automatically
- ✅ Build ROS2 workspace
- ✅ Verify installation
- ✅ Optionally start the system

Just run it and follow the prompts. Sudo will be requested only when needed.

---

## 📋 Manual Installation (5 steps)

If you prefer step-by-step control:

```bash
# 1. Generate configuration locally (interactive wizard - NO SUDO)
#    Creates bin/.venv and bin/secrets in current directory
bash bin/00_interactive_setup.sh

# 2. Setup host (creates /opt/obsea, copies config - REQUIRES SUDO)
#    Copies bin/.venv and bin/secrets to /opt/obsea/bin/
sudo bash bin/02_host_setup.sh

# 3. Install Python dependencies
sudo bash bin/12_install_python_deps.sh

# 4. Build ROS2 workspace
bash bin/30_build_ros2_ws.sh

# 5. Start system
bash bin/50_run_all.sh
```

Verify:
- API: `curl http://localhost:8101/health`
- Web: http://localhost:8180
- Docs: http://localhost:8101/docs

---

## 📋 Script Reference

| Script | Purpose | Requires sudo | Duration |
|--------|---------|---------------|----------|
| `00_interactive_setup.sh` | Generate config (answers questions) | No | 2 min |
| `02_host_setup.sh` | Create dirs, copy code, set permissions | **Yes** | 1 min |
| `03_create_users.sh` | Create operator users (optional) | **Yes** | per user |
| `12_install_python_deps.sh` | Install Python packages + venv | **Yes** | 5 min |
| `30_build_ros2_ws.sh` | Build ROS2 workspace (colcon build) | No | 10 min |
| `50_run_all.sh` | Start all services (runs forever) | No | ∞ |

---

## 🔧 Configuration

After `00_interactive_setup.sh`, config is generated in **local** `bin/.venv` (then copied to `/opt/obsea/bin/.venv` by `02_host_setup.sh`):

```bash
# System
HOSTNAME="obsea3-02"
OBSEA_MODE="dummy"             # prod or dummy
ROS_DOMAIN_ID="0"              # ROS isolation (0-232)

# Ports
UVICORN_PORT="8101"            # API endpoint
WEB_PORT="8180"                # Web frontend

# Hardware
SNMP_ENABLE="true"             # Network switch monitoring
SNMP_HOST="192.168.1.35"

# Monitoring
ZABBIX_ENABLED="true"
ZABBIX_HOST="192.168.1.50"
ZABBIX_PORT="10051"

# Paths (auto-configured)
BASE="/opt/obsea"
WS_MAIN="/opt/obsea/ros2_ws"
VENV_BASE="/opt/obsea/venv"
OBSEA3_DATA="/opt/obsea/data"
OBSEA_LOG_ROOT="/opt/obsea/log"
```

To change config: Edit `bin/.venv` in repository and re-run `sudo bash bin/02_host_setup.sh` to copy to `/opt/obsea`, or edit `/opt/obsea/bin/.venv` directly on the target system.

---

## 🔐 Secrets

Sensitive values (JWT secret, API tokens) are auto-generated in **local** `bin/secrets` by `00_interactive_setup.sh`, then copied to `/opt/obsea/bin/secrets` by `02_host_setup.sh`:

```bash
# After running 02_host_setup.sh, edit secrets:
sudo nano /opt/obsea/bin/secrets

# Or keep auto-generated ones from 00_interactive_setup.sh
cat /opt/obsea/bin/secrets
```

**Never commit `bin/secrets` to git** (already in `.gitignore`). The local `bin/secrets` is only used during deployment - production secrets live in `/opt/obsea/bin/secrets`.

---

## 📂 Directory Structure

```
/opt/obsea/
├── bin/                   # Scripts & config
│   ├── .venv              # Environment config (auto-generated)
│   ├── secrets            # Sensitive vars (git-ignored)
│   └── *.sh               # All deployment scripts
├── ros2_ws/               # ROS2 workspace
│   ├── src/               # Source packages (copied from repo)
│   ├── build/             # Build artifacts (colcon)
│   └── install/           # Installed packages
├── venv/ros2/             # Python virtualenv
├── data/                  # Runtime YAML configs, SQLite DBs
├── log/                   # Timestamped logs
│   └── current/ → YYYY/MM/DD/
└── archive/               # Config backups
```

---

## 🛠️ Common Tasks

### Service Management (systemd)

After installation, OBSEA3 runs as persistent systemd services with auto-start on boot:

```bash
# Ver si los servicios están habilitados (enabled) para arranque automático
sudo systemctl list-unit-files 'obsea3-*'

# Ver si los servicios están corriendo (active)
sudo systemctl is-active obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Ver estado detallado de todos los servicios
sudo systemctl status 'obsea3-*'

# Habilitar arranque automático de todos los servicios (uno por uno)
sudo systemctl enable obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Arrancar todos los servicios
sudo systemctl start obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Reiniciar todos los servicios
sudo systemctl restart obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Parar todos los servicios
sudo systemctl stop obsea3-hw-supervisor obsea3-api-gateway obsea3-web obsea3-zabbix

# Parar/arrancar servicios individuales
sudo systemctl stop obsea3-hw-supervisor
sudo systemctl start obsea3-api-gateway
sudo systemctl restart obsea3-web

# Ver logs en tiempo real
journalctl -u obsea3-hw-supervisor -f
tail -f /opt/obsea/log/current/hw_supervisor.log
```

**Available services:**
- `obsea3-hw-supervisor` - Hardware control node (ROS2)
- `obsea3-api-gateway` - FastAPI + ROS2 bridge
- `obsea3-web` - Vite dev server (React frontend)

All services:
- ✅ Auto-start on boot
- ✅ Auto-restart on failure
- ✅ Run in background (daemon mode)
- ✅ Log to `/opt/obsea/log/current/`

**View logs**:
```bash
tail -f /opt/obsea/log/current/api_gateway.log
tail -f /opt/obsea/log/current/hw_supervisor.log
tail -f /opt/obsea/log/current/zabbix.log
```

**Check health**:
```bash
curl http://localhost:8101/health
curl http://localhost:8101/docs    # API documentation
```

**Stop services (legacy manual mode)**:
```bash
# If foreground: Ctrl+C
# If background:
pkill -f ros2 && pkill -f uvicorn
```

**Rebuild after code changes**:
```bash
sudo rsync -a ~/obsea/ros2_ws/src/ /opt/obsea/ros2_ws/src/  # Update code
bash bin/30_build_ros2_ws.sh                                 # Rebuild
bash bin/50_run_all.sh                                       # Restart
```

**Create operator users**:
```bash
sudo bash bin/03_create_users.sh
```

**Default API credentials** (created automatically):
- Username: `test`
- Password: `test123`
- Role: `viewer` (read-only, cannot control ports)

Create admin users: `python3 bin/more/obsea_user_admin.py`

---

## ⚠️ Troubleshooting

**"No packages found" during build**:
```bash
sudo rsync -a ~/obsea/ros2_ws/src/ /opt/obsea/ros2_ws/src/
bash bin/30_build_ros2_ws.sh
```

**"Missing config file" error** (e.g., zabbix.yaml):
```bash
# Rebuild to reinstall config files
bash bin/30_build_ros2_ws.sh
```

**API won't start / port in use**:
```bash
lsof -i :8101           # Check what's using port
pkill -f uvicorn        # Kill old process
bash bin/50_run_all.sh  # Restart
```

**Permission denied**:
```bash
chmod +x bin/*.sh                    # Make scripts executable
sudo bash bin/02_host_setup.sh       # Re-fix permissions
```

---

## 📚 Documentation

- **Scripts**: `bin/README.md` (all scripts explained)
- **Architecture**: `.github/ai-agent-instructions.md` (technical details)

---

## ✅ Pre-Launch Checklist

- [ ] Generated config: `bash bin/00_interactive_setup.sh`
- [ ] Set up host: `sudo bash bin/02_host_setup.sh`
- [ ] Edited secrets: `/opt/obsea/bin/secrets` with real values
- [ ] Installed deps: `sudo bash bin/12_install_python_deps.sh`
- [ ] Built workspace: `bash bin/30_build_ros2_ws.sh` (no errors)
- [ ] Started system: `bash bin/50_run_all.sh`
- [ ] API health check: `curl http://localhost:8101/health` returns OK
- [ ] Web loads: http://localhost:8180

---

**Questions?** Check logs in `/opt/obsea/log/current/` or see `bin/README.md`.
