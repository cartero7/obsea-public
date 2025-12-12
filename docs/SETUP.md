# Configuración Inicial OBSEA3

> ⚠️ **NOTA**: Este documento describe el proceso manual paso a paso.  
> Para instalación automática usa: `bash bin/00_install_all.sh`  
> Ver [INSTALL.md](../INSTALL.md) para guía rápida.

Esta guía describe los pasos para configurar OBSEA3 desde cero en una instalación nueva.

## Requisitos Previos

- **Sistema Operativo**: Ubuntu 24.04 LTS (Noble Numbat) - requerido para ROS2 Jazzy
- **ROS2**: Jazzy Jalisco instalado en `/opt/ros/jazzy`
- **Python**: 3.12+ con pip
- **Hardware** (opcional): Sistema embebido con I2C habilitado para modo producción
- **Acceso**: Usuario con sudo

## Instalación Completa (Pasos Secuenciales)

### 1. Configuración Interactiva Inicial

```bash
cd /opt/obsea
bash bin/00_interactive_setup.sh
```

**Acciones**:
- Solicita información del sistema (hostname, modo, dominio ROS2)
- Genera archivo `bin/.venv` con variables de entorno
- Genera archivo `bin/secrets` con tokens de autenticación
- Crea estructura de directorios básica

**Preguntas interactivas**:
```
Hostname [obsea3]: <ENTER o nombre personalizado>
Modo [prod/dummy] (dummy): prod
ROS Domain ID [0-232] (42): <ENTER o ID personalizado>
Puerto API [8101]: <ENTER>
Puerto Web [8180]: <ENTER>
```

**Resultado esperado**:
```
✓ Archivo bin/.venv creado
✓ Archivo bin/secrets creado
✓ Directorios data/, log/, archive/ creados
```

### 2. Configuración del Sistema (Requiere sudo)

```bash
sudo bash bin/02_host_setup.sh
```

**Acciones**:
- Crea usuario del sistema: `obsea3` (si no existe)
- Crea grupo: `obsea3`
- Configura permisos de directorios: `/opt/obsea`
- Configura I2C (añade usuario a grupo `i2c`)
- Copia systemd services (opcional)

**Verificación**:
```bash
# Usuario creado
id obsea3

# Permisos correctos
ls -ld /opt/obsea
# Esperado: drwxrwxr-x obsea3 obsea3

# Acceso I2C
groups | grep i2c
```

### 3. Instalar Dependencias del Sistema

```bash
sudo bash bin/10_install_system_deps.sh
```

**Acciones**:
- Instala paquetes del sistema necesarios:
  - `build-essential`, `cmake`, `git`
  - `python3-dev`, `python3-pip`
  - `i2c-tools`, `libi2c-dev`
  - `sqlite3`, `libsqlite3-dev`
  - `snmp`, `libsnmp-dev`
  - `jq`, `curl`, `wget`
- Verifica instalación de ROS2 Jazzy

**Verificación**:
```bash
# Verificar ROS2
source /opt/ros/jazzy/setup.bash
ros2 --version
# Esperado: ros2 doctor version X.Y.Z

# Verificar I2C
i2cdetect -l
# Esperado: lista de buses i2c (si hardware presente)

# Verificar snmp
snmpget --version
```

### 4. Instalar Dependencias de Python

```bash
sudo bash bin/12_install_python_deps.sh
```

**Acciones**:
- Crea virtualenv en `/opt/obsea/venv/ros2`
- Instala paquetes Python:
  - `fastapi`, `uvicorn[standard]`
  - `pyjwt`, `passlib[bcrypt]`
  - `pyyaml`, `python-multipart`
  - `smbus2`, `adafruit-circuitpython-*`
  - `pytest`, `pytest-cov` (desarrollo)

**Verificación**:
```bash
source venv/ros2/bin/activate
pip list | grep fastapi
pip list | grep smbus2
deactivate
```

### 5. Crear Usuarios de la Aplicación

```bash
bash bin/03_create_users.sh
```

**Acciones**:
- Crea base de datos SQLite: `/opt/obsea/data/auth.db`
- Ejecuta migraciones de esquema
- Crea usuarios iniciales:
  - `admin` (role: admin, password: admin123)
  - `operator` (role: operator, password: operator123)
  - `test` (role: viewer, password: test123)

**Personalizar usuarios**:
```bash
# Usar script interactivo
python bin/more/obsea_user_admin.py

# Comandos disponibles
python bin/more/obsea_user_admin.py --list
python bin/more/obsea_user_admin.py --add username --role admin
python bin/more/obsea_user_admin.py --delete username
python bin/more/obsea_user_admin.py --reset-password username
```

**Verificación**:
```bash
sqlite3 /opt/obsea/data/auth.db << EOF
SELECT username, role FROM users;
EOF
# Esperado: admin, operator, test
```

### 6. Compilar Workspace ROS2

```bash
bash bin/30_build_ros2_ws.sh
```

**Acciones**:
- Verifica ROS2 sourced
- Ejecuta colcon build en `ros2_ws/`
- Usa `--merge-install` para instalación única
- Tipo de build: `RelWithDebInfo` (configurable)

**Tiempo esperado**: 30-60 segundos primera vez, 3-5 segundos subsecuentes.

**Verificación**:
```bash
source ros2_ws/install/setup.bash
ros2 pkg list | grep obsea3
# Esperado:
# obsea3_api_gateway
# obsea3_core
# obsea3_hw_interfaces
# obsea3_interfaces
# obsea3_web
```

### 7. Inicializar Base de Datos de la API

```bash
bash bin/20_db_migrate_obsea_api.sh
```

**Acciones**:
- Crea base de datos: `/opt/obsea/data/obsea3_api.db`
- Ejecuta migraciones en `obsea3_api_gateway/migrations/`
- Crea tablas: `system_events`, `port_actions`, `config_changes`, etc.

**Verificación**:
```bash
sqlite3 /opt/obsea/data/obsea3_api.db << EOF
.tables
EOF
# Esperado: system_events, port_actions, ...
```

### 8. Copiar Configuraciones Iniciales

```bash
# Copiar templates a runtime
cp ros2_ws/src/obsea3_core/config/*.yaml data/

# Verificar
ls -lh data/*.yaml
# Esperado: instruments.yaml, limits.yaml, backplane.yaml, calibrations.yaml
```

### 9. Primera Ejecución (Testing)

```bash
bash bin/50_run_all.sh
```

**Servicios iniciados**:
- Hardware Supervisor (puerto I2C o dummy)
- API Gateway (http://localhost:8101)
- Web Frontend (http://localhost:8180)
- Zabbix Bridge (si habilitado)

**Verificación**:
```bash
# Verificar procesos
ps aux | grep obsea3

# Verificar logs
tail -f log/current/api_gateway.log
tail -f log/current/hw_supervisor.log

# Verificar API
curl http://localhost:8101/health
# Esperado: {"ok": true, "node": "api_gateway"}

# Verificar nodos ROS2
source ros2_ws/install/setup.bash
ros2 node list
# Esperado: /hw_supervisor, /api_gateway, ...
```

## Configuración Avanzada

### Modo Producción vs Dummy

**Modo Dummy** (desarrollo sin hardware):
```bash
# En bin/.venv
OBSEA_MODE=dummy
```

**Modo Producción** (con hardware real):
```bash
# En bin/.venv
OBSEA_MODE=prod

# Verificar I2C disponible
i2cdetect -y 1
# Debe mostrar dispositivos en direcciones esperadas
```

### Variables de Entorno Importantes

Editar `bin/.venv`:
```bash
# Identificación
export HOSTNAME=obsea3
export OBSEA_MODE=dummy  # o 'prod'

# ROS2
export ROS_DOMAIN_ID=42  # 0-232
export OBSEA3_NS=/obsea3

# Puertos
export UVICORN_PORT=8101
export WEB_PORT=8180

# Rutas
export BASE=/opt/obsea
export WS_MAIN=$BASE/ros2_ws
export VENV_BASE=$BASE/venv/ros2
export LOG_DIR=$BASE/log

# Features
export DRY_RUN=false
export SNMP_ENABLE=true
export ZABBIX_ENABLED=false
```

### Tokens de Autenticación

Editar `bin/secrets`:
```bash
# JWT para API
export OBSEA3_JWT_SECRET=$(openssl rand -base64 32)

# Token para comandos hardware
export OBSEA3_TOKEN=$(openssl rand -base64 32)
```

**IMPORTANTE**: Nunca commits `bin/secrets` a Git. Usar `bin/secrets.example` como template.

### Configurar Límites de Seguridad

Editar `data/limits.yaml`:
```yaml
ports:
  1:
    max_current: 2.0      # Amperios
    min_voltage: 11.0     # Voltios
    max_voltage: 13.0
    overcurrent_duration: 0.5  # Segundos
  2:
    max_current: 1.0
    min_voltage: 11.0
    max_voltage: 13.0
    overcurrent_duration: 0.2

buses:
  v12:
    max_current: 10.0
    min_voltage: 11.5
    max_voltage: 12.5
  v48:
    max_current: 5.0
    min_voltage: 47.0
    max_voltage: 49.0
```

**Hot-reload**: Hardware supervisor recarga automáticamente cada 5 segundos.

### Configurar Instrumentos

Editar `data/instruments.yaml`:
```yaml
ports:
  1:
    name: "CTD SBE37"
    type: "CTD"
    params:
      serial_port: "/dev/ttyUSB0"
      baud_rate: 9600
  2:
    name: "Camera IP 1"
    type: "CAMERA_IP"
    params:
      ip_address: "192.168.1.100"
      username: "admin"
      password_file: "/opt/obsea/data/camera1_pass.txt"
```

### Calibraciones de Sensores

Editar `data/calibrations.yaml`:
```yaml
ports:
  1:
    voltage_offset: 0.05
    voltage_scale: 1.001
    current_offset: 0.02
    current_scale: 0.998
  2:
    voltage_offset: 0.03
    voltage_scale: 1.002
    current_offset: 0.01
    current_scale: 0.999
```

## Verificación Post-Instalación

### Checklist Completo

```bash
# 1. Variables de entorno
set -a; source bin/.venv; set +a
echo $HOSTNAME $OBSEA_MODE $ROS_DOMAIN_ID
# Esperado: valores configurados

# 2. Secrets cargados
source bin/secrets
echo ${OBSEA3_JWT_SECRET:0:10}...  # Primeros 10 caracteres
# Esperado: string base64

# 3. Workspace compilado
source ros2_ws/install/setup.bash
ros2 pkg list | grep obsea3 | wc -l
# Esperado: 5

# 4. Bases de datos
ls -lh data/*.db
# Esperado: auth.db, obsea3_api.db

# 5. Configuraciones
ls -lh data/*.yaml | wc -l
# Esperado: 4 (instruments, limits, backplane, calibrations)

# 6. Permisos
ls -ld /opt/obsea
# Esperado: drwxrwxr-x obsea3 obsea3

# 7. Python deps
source venv/ros2/bin/activate
pip list | grep -E "(fastapi|smbus2|pyjwt)" | wc -l
# Esperado: ≥3
deactivate

# 8. Sistema corriendo
curl -s http://localhost:8101/health | jq .ok
# Esperado: true
```

### Test de Login API

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:8101/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' \
  | jq -r .access_token)

# Verificar token
echo $TOKEN | cut -d'.' -f1 | base64 -d 2>/dev/null | jq
# Esperado: header JWT con alg y typ

# Usar token
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8101/ports/1/telemetry | jq .port
# Esperado: 1
```

### Test de Comando de Puerto

```bash
source ros2_ws/install/setup.bash
source bin/secrets

# Comando ROS2
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$OBSEA3_TOKEN'}"

# Esperado:
# success: true
# message: "Port 1 set to 12V"
```

### Test de Alarmas

```bash
# Consultar alarmas activas
bash bin/more/check_alarms.sh

# Esperado (si todo OK):
# 0 alarma(s) detectada(s) en las últimas 24 horas
```

## Troubleshooting Instalación

### Error: "ROS2 not found"
```bash
# Verificar instalación
ls /opt/ros/jazzy/setup.bash

# Si no existe, instalar ROS2 Jazzy
# Seguir: https://docs.ros.org/en/jazzy/Installation.html
```

### Error: "Permission denied" en I2C
```bash
# Añadir usuario a grupo i2c
sudo usermod -a -G i2c $(whoami)

# Logout y login de nuevo
```

### Error: "Port 8101 already in use"
```bash
# Buscar proceso
sudo lsof -i :8101

# Detener
pkill uvicorn

# O cambiar puerto en bin/.venv
export UVICORN_PORT=8102
```

### Error: "Database locked"
```bash
# Verificar procesos usando DB
lsof /opt/obsea/data/obsea3_api.db

# Detener servicios
pkill -f obsea3

# Reiniciar
bash bin/50_run_all.sh
```

### Error: Build falla con colcon
```bash
# Limpiar build cache
rm -rf ros2_ws/build ros2_ws/install ros2_ws/log

# Verificar permisos
ls -ld ros2_ws/
# Debe ser owned by usuario actual, no root

# Rebuild
bash bin/30_build_ros2_ws.sh
```

## Próximos Pasos

Después de la instalación exitosa:

1. **Personalizar configuraciones**: Editar `data/*.yaml` según hardware real
2. **Crear usuarios adicionales**: `python bin/more/obsea_user_admin.py`
3. **Configurar systemd** (producción): Ver `bin/archived/README.md`
4. **Integrar Zabbix**: Configurar en `bin/.venv` → `ZABBIX_ENABLED=true`
5. **Desarrollar instrumentos**: Añadir drivers en `obsea3_core/instruments/`

## Referencias

- **Desarrollo**: `docs/DESARROLLO.md`
- **Alarmas**: `docs/ALARMAS.md`
- **Arquitectura**: `.github/ARCHITECTURE.md`
- **Scripts**: `bin/README.md`
