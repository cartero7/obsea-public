# Guía de Desarrollo OBSEA3

## Ciclo de Desarrollo Diario

### 1. Preparar Entorno
```bash
cd /opt/obsea

# Verificar variables de entorno
set -a
source bin/.venv
set +a

# Verificar valores clave
echo "HOSTNAME: $HOSTNAME"
echo "OBSEA_MODE: $OBSEA_MODE"
echo "ROS_DOMAIN_ID: $ROS_DOMAIN_ID"
echo "BASE: $BASE"
```

### 2. Hacer Cambios en Código
```bash
# Editar archivos en ros2_ws/src/
code ros2_ws/src/obsea3_hw_interfaces/obsea3_hw_interfaces/hw_supervisor_node.py
code ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/main.py
```

### 3. Compilar Workspace
```bash
# Compilación completa
bash bin/30_build_ros2_ws.sh

# Verificar resultado
echo $?  # 0 = éxito, !0 = error

# Ver logs de build
tail -100 ros2_ws/log/latest_build/events.log
```

**Tiempo esperado**: 3-5 segundos para workspace limpio.

### 4. Probar Cambios

#### Opción A: Iniciar Todo el Sistema
```bash
bash bin/50_run_all.sh
```

Esto inicia:
- Hardware supervisor (hw_supervisor_node)
- API Gateway (uvicorn en puerto 8101)
- Web frontend (servidor estático en puerto 8180)
- Zabbix bridge (si está habilitado)

#### Opción B: Iniciar Componentes Individuales

**Solo Hardware Supervisor**:
```bash
source ros2_ws/install/setup.bash
ros2 run obsea3_hw_interfaces hw_supervisor_node --ros-args -p dummy:=true
```

**Solo API Gateway (modo desarrollo con hot-reload)**:
```bash
set -a; source bin/.venv; set +a
python -m uvicorn obsea3_api_gateway.main:app --reload --port 8101
```

**Solo Zabbix Bridge**:
```bash
source ros2_ws/install/setup.bash
ros2 run obsea3_core zabbix_bridge_node
```

### 5. Monitorear Logs
```bash
# API Gateway
tail -f log/current/api_gateway.log

# Hardware Supervisor
tail -f log/current/hw_supervisor.log

# Zabbix Bridge
tail -f log/current/zabbix_bridge.log

# Todos simultáneamente
multitail log/current/*.log
```

### 6. Probar Funcionalidad

#### Verificar Nodos ROS2
```bash
source ros2_ws/install/setup.bash
ros2 node list
# Esperado: /api_gateway, /hw_supervisor, /safety_manager, /zabbix_bridge

ros2 topic list
# Esperado: /obsea3/port/1/telemetry, /obsea3/rails/v12, etc.
```

#### Verificar API REST
```bash
# Health check
curl http://localhost:8101/health
# Esperado: {"ok": true, "node": "api_gateway"}

# Login
TOKEN=$(curl -s -X POST http://localhost:8101/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' \
  | jq -r .access_token)

# Telemetría puerto 1
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8101/ports/1/telemetry | jq
```

#### Verificar Comandos de Puerto
```bash
source ros2_ws/install/setup.bash

# Leer token desde secrets
source bin/secrets

# Comando vía ROS2 (requiere token)
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$OBSEA3_TOKEN'}"

# Comando vía API REST
curl -X POST http://localhost:8101/ports/1/set_mode \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode": "12V"}'
```

### 7. Detener Servicios
```bash
# Detener todos los procesos Python
pkill -f obsea3
pkill -f uvicorn

# O usar Ctrl+C en cada terminal si lanzaste manualmente
```

## Agregar Nueva Funcionalidad

### Caso 1: Nuevo Mensaje ROS2

1. **Definir mensaje**:
```bash
cd ros2_ws/src/obsea3_interfaces/msg
cat > NewTelemetry.msg << EOF
float64 temperature
float64 pressure
int64 timestamp
EOF
```

2. **Actualizar CMakeLists.txt**:
```bash
cd ros2_ws/src/obsea3_interfaces
# Editar CMakeLists.txt
nano CMakeLists.txt
# Agregar "msg/NewTelemetry.msg" a rosidl_generate_interfaces()
```

3. **Compilar**:
```bash
cd /opt/obsea
bash bin/30_build_ros2_ws.sh
```

4. **Usar en Python**:
```python
from obsea3_interfaces.msg import NewTelemetry

# Publicar
msg = NewTelemetry()
msg.temperature = 15.5
msg.pressure = 1013.25
msg.timestamp = self.get_clock().now().nanoseconds
publisher.publish(msg)

# Suscribir
def callback(msg):
    print(f"Temp: {msg.temperature}, Press: {msg.pressure}")
subscription = self.create_subscription(NewTelemetry, '/topic', callback, 10)
```

### Caso 2: Nuevo Endpoint API

1. **Agregar ruta en main.py**:
```python
# ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/main.py

@app.get("/custom/data")
async def get_custom_data(user: dict = Depends(require_user)):
    """Endpoint de ejemplo con autenticación."""
    # Lógica aquí
    return {"data": "example"}
```

2. **Agregar función de negocio** (si es compleja):
```python
# ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/custom_logic.py

def process_custom_data():
    """Lógica de negocio separada."""
    return {"processed": True}
```

3. **Importar y usar**:
```python
# En main.py
from .custom_logic import process_custom_data

@app.get("/custom/data")
async def get_custom_data(user: dict = Depends(require_user)):
    result = process_custom_data()
    return result
```

4. **Recargar API** (con --reload activo):
```bash
# Automático si usaste uvicorn con --reload
# Manual:
pkill uvicorn
bash bin/obsea3_api_gateway_run.sh
```

5. **Probar**:
```bash
TOKEN=$(curl -s -X POST http://localhost:8101/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' \
  | jq -r .access_token)

curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8101/custom/data | jq
```

### Caso 3: Nueva Configuración YAML

1. **Crear archivo de configuración**:
```bash
cat > ros2_ws/src/obsea3_core/config/new_config.yaml << EOF
settings:
  param1: value1
  param2: 42
  nested:
    option_a: true
    option_b: false
EOF
```

2. **Copiar a runtime**:
```bash
cp ros2_ws/src/obsea3_core/config/new_config.yaml /opt/obsea/data/
```

3. **Cargar en código**:
```python
from obsea3_api_gateway.config_store import load_yaml, save_yaml

# Cargar
config = load_yaml("new_config")  # Busca en /opt/obsea/data/

# Usar
param1 = config["settings"]["param1"]

# Modificar y guardar (auto-archiva)
config["settings"]["param2"] = 99
save_yaml("new_config", config)
```

4. **Verificar archivo en archive**:
```bash
ls -lh /opt/obsea/archive/config/new_config/$(date +%Y/%m/)
# Verás: new_config.YYYYMMDDTHHMMSSZ.yaml
```

### Caso 4: Nueva Tabla en Base de Datos

1. **Crear migración** (estilo Alembic):
```bash
cd ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/migrations
cat > 003_add_custom_table.py << 'EOF'
"""
Migration: Add custom_table
Date: 2025-12-09
"""

def upgrade(conn):
    conn.execute("""
        CREATE TABLE IF NOT EXISTS custom_table (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            data TEXT,
            port INTEGER
        )
    """)
    conn.execute("CREATE INDEX IF NOT EXISTS idx_custom_timestamp ON custom_table(timestamp)")

def downgrade(conn):
    conn.execute("DROP TABLE IF EXISTS custom_table")
EOF
```

2. **Ejecutar migración**:
```bash
bash bin/20_db_migrate_obsea_api.sh
# Verifica logs: log/current/db_migrate_api.log
```

3. **Usar en código**:
```python
from obsea3_api_gateway.actions_log import get_db_connection

def insert_custom_data(data, port):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO custom_table (timestamp, data, port) VALUES (?, ?, ?)",
        (datetime.utcnow().isoformat(), data, port)
    )
    conn.commit()
    conn.close()
```

4. **Rollback si es necesario**:
```bash
DOWNGRADE=1 bash bin/20_db_migrate_obsea_api.sh
```

## Modo Dummy para Testing

### Activar Modo Dummy
```bash
# Método 1: Editar .venv
nano bin/.venv
# Cambiar: OBSEA_MODE=dummy

# Método 2: Parámetro launch
source ros2_ws/install/setup.bash
ros2 run obsea3_hw_interfaces hw_supervisor_node --ros-args -p dummy:=true
```

### Comportamiento en Modo Dummy
- **I2C**: Genera datos sintéticos aleatorios
  - Voltajes: 11.5-12.5V (12V), 47-49V (48V)
  - Corrientes: 0.1-2.0A por puerto
  - Temperatura: 15-25°C
- **SNMP**: Retorna datos sintéticos o ceros (según `dummy_snmp_policy`)
- **Estado**: Persiste en `last_state_file` como en modo real
- **Comandos**: Funcionan normalmente (cambios de modo OFF/12V/48V)

### Probar sin Hardware
```bash
# Terminal 1: HW Supervisor en dummy
source ros2_ws/install/setup.bash
ros2 run obsea3_hw_interfaces hw_supervisor_node --ros-args -p dummy:=true

# Terminal 2: Monitor telemetría
source ros2_ws/install/setup.bash
ros2 topic echo /obsea3/port/1/telemetry

# Terminal 3: Enviar comando
source ros2_ws/install/setup.bash
source bin/secrets
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$OBSEA3_TOKEN'}"
```

## Errores Comunes y Soluciones

### Build falla con "Permission denied"
**Causa**: Usuario incorrecto en `.venv` o permisos de archivos.

**Solución**:
```bash
# Verificar usuario
cat bin/.venv | grep USER_NAME
# Debe coincidir con: whoami

# Arreglar permisos
sudo chown -R $(whoami):$(whoami) /opt/obsea/ros2_ws
```

### "Could not find package obsea3_interfaces"
**Causa**: ROS2 underlay no sourced.

**Solución**:
```bash
source /opt/ros/jazzy/setup.bash
source ros2_ws/install/setup.bash
```

### "Invalid token" en comandos
**Causa**: Token en `bin/secrets` no coincide con el esperado.

**Solución**:
```bash
# Verificar token
cat bin/secrets | grep OBSEA3_TOKEN

# Regenerar token
openssl rand -base64 32 > /tmp/new_token
echo "OBSEA3_TOKEN=$(cat /tmp/new_token)" >> bin/secrets
```

### API devuelve 404 en endpoints
**Causa**: Ruta incorrecta o API no iniciada.

**Verificación**:
```bash
# Verificar proceso
ps aux | grep uvicorn

# Verificar logs
tail -30 log/current/api_gateway.log

# Verificar rutas disponibles
curl http://localhost:8101/docs  # OpenAPI UI
```

### Nodos ROS2 no se descubren
**Causa**: `ROS_DOMAIN_ID` diferente entre terminales.

**Solución**:
```bash
# En cada terminal
echo $ROS_DOMAIN_ID  # Debe ser el mismo valor (0-232)

# Forzar mismo domain
export ROS_DOMAIN_ID=42
```

## Testing

### Tests Rápidos
```bash
bash bin/quick_test.sh
```

Verifica:
- Interfaces ROS2 compiladas
- Configuraciones YAML cargables
- Imports de Python

### Tests Comprehensivos
```bash
bash bin/comprehensive_test.sh
```

Requiere sistema corriendo. Verifica:
- Nodos ROS2 activos
- API endpoints responden
- Base de datos accesible
- Comandos funcionan

### Tests Unitarios (Pytest)
```bash
# Instalar pytest si no está
pip install pytest

# Ejecutar tests de módulo
cd ros2_ws/src/obsea3_api_gateway
pytest tests/

# Con coverage
pytest --cov=obsea3_api_gateway tests/
```

## Diagnóstico Remoto

### Recolectar Estado del Sistema
```bash
bash bin/remote_diagnostics.sh
```

Genera archivo: `obsea_diagnostics_YYYYMMDD_HHMMSS.tar.gz`

Contiene:
- Logs recientes
- Configuraciones
- Estado ROS2
- Procesos activos
- Estado base de datos

### Analizar Diagnóstico
```bash
tar -tzf obsea_diagnostics_*.tar.gz  # Listar contenido
tar -xzf obsea_diagnostics_*.tar.gz  # Extraer
cd diagnostics/
cat system_info.txt
cat ros2_info.txt
```

## Referencias

- **Scripts de desarrollo**: `bin/README.md`
- **Arquitectura**: `.github/ARCHITECTURE.md`
- **Alarmas**: `docs/ALARMAS.md`
- **Usuarios**: `bin/more/obsea_user_admin.py --help`
