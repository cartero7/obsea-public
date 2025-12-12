# OBSEA3 - API & ROS2 Usage Examples

> **Public documentation repository** for OBSEA3 submarine power & instrument control system.

🚀 **Quick Start**: Browse the examples below  
📧 **Contact**: For full source code access, contact project maintainers

---

## 📚 Documentation

### 🌐 REST API Examples
→ **[docs/API_EJEMPLOS.md](docs/API_EJEMPLOS.md)**

Complete REST API usage guide with real-world examples:
- Authentication & JWT tokens
- Port control (12V/48V switching)
- Real-time telemetry & monitoring
- System configuration
- Lease management (exclusive port control)
- Python & curl examples

### 🤖 ROS2 Integration Examples
→ **[docs/ROS2_EJEMPLOS.md](docs/ROS2_EJEMPLOS.md)**

ROS2 service integration guide with automation examples:
- ROS2 service calls with JWT authentication
- Direct hardware control from ROS2 nodes
- Port control automation scripts
- System token vs user authentication
- Bash script examples for common tasks

---

## 🎯 System Overview

OBSEA3 controls submarine power rails (12V/48V) for underwater instruments via I2C sensors.

**Architecture:**
- **Backend**: ROS2 Jazzy + FastAPI gateway
- **Hardware**: I2C sensors (LTC2945, MCP23008, ADS1115)
- **Frontend**: React + Vite responsive web interface
- **Security**: JWT authentication with role-based access control
- **Safety**: Real-time monitoring with automatic protection limits

**Key Features:**
- 🔌 12V/48V power rail switching
- 🔐 Multi-user authentication with port-level permissions
- 📊 Real-time telemetry & alarm system
- 🚨 Overcurrent/undervolt protection with auto-shutdown
- 📱 Mobile-responsive web interface
- 🔧 ROS2 integration for research workflows

---

## 📜 License

See [LICENSE](LICENSE) file.

## 👤 Credits

**Designed by**: Carla Artero Delgado (UPC) — Catalunya  
**Project**: OBSEA Submarine Observatory
