# OBSEA3 — Control de Potencia e Instrumentos Submarinos

Sistema basado en ROS2 con backend FastAPI y frontend Vite/React opcional.

```
╔════════════════════════════════════════════════════════════════╗
║          🌊 Arquitectura del Sistema OBSEA3 🌊                 ║
╚════════════════════════════════════════════════════════════════╝

   ┌──────────────────────────────────────────────────────────┐
   │  🌐 Interfaz Web (React + Vite)                          │
   │  ├─ Dashboard & Telemetría en Tiempo Real                │
   │  ├─ Control de Puertos (conmutación 12V/48V)             │
   │  └─ Gestión de Usuarios y Editor de Config               │
   └──────────────────────┬───────────────────────────────────┘
                          │ HTTP/REST + WebSocket
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  🚀 API Gateway (FastAPI)                                │
   │  ├─ Autenticación JWT & Bloqueos de Puertos              │
   │  ├─ Registro de Auditoría & Comprobaciones               │
   │  └─ Puente ROS2 (Topics ↔ REST/WS)                       │
   └──────────────────────┬───────────────────────────────────┘
                          │ Servicios y Topics ROS2
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  🤖 Supervisor de Hardware (ROS2 Jazzy)                  │
   │  ├─ Sensores I2C (LTC2945, MCP23008, ADS1115)            │
   │  ├─ Monitoreo de Red SNMP                                │
   │  ├─ Gestor de Seguridad (Límites V/I)                    │
   │  └─ Máquina de Estados & Sistema de Alarmas              │
   └──────────────────────┬───────────────────────────────────┘
                          │ Bus I2C + GPIO + SNMP
                          ↓
   ┌──────────────────────────────────────────────────────────┐
   │  ⚡ Backplane de Hardware (PCB Personalizado)             │
   │  └─ Rieles: 48V / 12V / 5V con monitoreo                 │
   └──────────────────────┬───────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
       ┌──────▼──────┐         ┌──────▼──────┐
       │ 🔌 Puerto1-4│   ...   │ 🔌 Puerto5-8│
       │  (12V/48V)  │         │  (12V/48V)  │
       └──────┬──────┘         └──────┬──────┘
              │                       │
              └───────────┬───────────┘
                          ↓
                 ┌──────────────────────┐
                 │ 🪼 Laboratorio       │
                 │    Submarino         │
                 │  ├─ Sensores CTD     │
                 │  ├─ Cámaras IP       │
                 │  ├─ Hidrófonos       │
                 │  └─ Instrumentos     │
                 └──────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │  📊 Monitoreo y Observabilidad                           │
   │  ├─ Puente Zabbix → 📈 Servidor de Métricas              │
   │  ├─ Registros SQLite (Acciones + Alarmas)                │
   │  └─ Versionado de Config (Archivos YAML)                 │
   └──────────────────────────────────────────────────────────┘
```

> **Diseñado por**: Carla Artero Delgado (UPC) — Catalunya  
> **Licencia**: Ver archivo LICENSE

> 🇬🇧 Versión en inglés: [README.en.md](README.en.md)

## Documentación

- `INSTALL.md` — Instalación rápida
- `DEPLOYMENT_GUIDE.md` — Instalación completa, configuración y troubleshooting
- `docs/WORKFLOW.md` — **Flujo de trabajo de desarrollo** (editar → compilar → probar → commit)
- `bin/README.md` — Referencia de scripts
- `.github/ARCHITECTURE.md` — Ficha rápida de arquitectura

## Instalación (5 pasos)

**Requisitos**: Ubuntu 24.04 LTS (Noble Numbat) - x86_64 o ARM64

```bash
# 1) Generar configuración (crea .venv y plantilla de secretos)
bash bin/00_interactive_setup.sh

# 2) Preparar host (crea dirs, copia código, ajusta permisos)
sudo bash bin/02_host_setup.sh

# 3) Instalar dependencias Python
sudo bash bin/12_install_python_deps.sh

# 4) Compilar workspace ROS2
bash bin/30_build_ros2_ws.sh

# 5) Iniciar sistema
bash bin/50_run_all.sh
```

Salud: `curl http://localhost:8101/health`

**Credenciales por defecto:**
- Usuario: `test`
- Contraseña: `test123`  
- Rol: `viewer` (solo lectura, sin control de puertos)

Crear usuarios admin: `python3 bin/more/obsea_user_admin.py`

## Estructura

- `ros2_ws/src/` — paquetes ROS2 (API gateway, nodos core, interfaces hardware, definiciones de mensajes, proxy web)
- `bin/` — scripts de despliegue/administración; legado en `bin/archived/`
- `webtest/` — frontend Vite/React (ignorado por defecto; quita `webtest/` de `.gitignore` si quieres versionarlo)
- `.github/ai-agent-instructions.md` — resumen de arquitectura

## Tareas comunes

- Validación rápida: `bash bin/quick_test.sh`
- Validación completa: `bash bin/comprehensive_test.sh`
- Gestión de usuarios (sudo): `bash bin/03_create_users.sh`
- Logs (host de runtime): `/opt/obsea/log/current/`
- Secretos: plantilla `bin/secrets.example`; secretos reales en `/opt/obsea/bin/secrets`

## Gestión de Servicios (systemd)

Después de la instalación, OBSEA3 funciona como servicios systemd persistentes:

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

**Servicios:**
- `obsea3-hw-supervisor` - Nodo de control hardware (ROS2)
- `obsea3-api-gateway` - FastAPI + puente ROS2
- `obsea3-web` - Servidor Vite (frontend React)

Todos los servicios se inician automáticamente al arrancar y se reinician automáticamente si fallan.

## Notas

- Se ignoran artefactos de build (`ros2_ws/build`, `ros2_ws/install`) y `webtest/node_modules`.
- Si necesitas versionar el frontend, elimina `webtest/` de `.gitignore`.

---

## Créditos y Licencia

**Diseño y Desarrollo del Sistema**:  
🧑‍💻 **Carla Artero Delgado**  
📍 Universitat Politècnica de Catalunya (UPC)  
🌍 Catalunya

**Proyecto**: Sistema de Control del Observatorio Submarino OBSEA3  
**Tecnologías**: ROS2 Jazzy, FastAPI, React, Python, I2C, SNMP

**Repositorio**: [github.com/cartero7/obsea](https://github.com/cartero7/obsea)

---

Hecho con 🪼 para la ciencia submarina
