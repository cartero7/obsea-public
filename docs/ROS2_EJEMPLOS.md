# OBSEA3 - Ejemplos de Uso con ROS2

Este documento muestra cómo interactuar con el sistema OBSEA3 usando **comandos ROS2** directamente desde la terminal (para desarrollo y debugging).

> 📘 **¿Buscas ejemplos de API REST para usuarios finales?** Ver [API_EJEMPLOS.md](API_EJEMPLOS.md) (recomendado)

## 🆕 Autenticación de Usuario en ROS2 (Nuevo)

**A partir de diciembre 2025**, el servicio ROS2 `/obsea3/set_port_mode` soporta **autenticación dual**:

1. **Token de sistema** (OBSEA3_TOKEN) → Solo debugging, acceso root
2. **JWT de usuario** (username + token) → **NUEVO**: Producción, con roles y permisos por puerto

Esto significa que ahora puedes usar `ros2 service call` con la **misma seguridad que la API REST**:
- ✅ Validación de JWT (firma + expiración)
- ✅ Roles (admin, operator, viewer)
- ✅ Permisos por puerto (user_ports tabla)
- ✅ Leases (control exclusivo)

**Ver sección [2b. Cambiar Modo con Autenticación de Usuario](#2b-cambiar-modo-con-autenticación-de-usuario-ros2--jwt)** para ejemplos completos.

## Configuración Inicial

Antes de ejecutar cualquier comando ROS2, **SIEMPRE** configurar el entorno:

### Paso 1: Ver el ROS_DOMAIN_ID actual del sistema

```bash
# Ver el ROS_DOMAIN_ID que usan los servicios
grep ROS_DOMAIN_ID /opt/obsea/bin/obsea3.env

# O consultando directamente el .venv
source /opt/obsea/bin/.venv && echo "ROS_DOMAIN_ID=$ROS_DOMAIN_ID"
```

**Valores típicos por hostname:**
- `obsea3-01` → ROS_DOMAIN_ID=77 (producción)
- `obsea3-02` o `obsea3-03` → ROS_DOMAIN_ID=5 (desarrollo/dummy)
- Otros sistemas → ROS_DOMAIN_ID=5 (por defecto)

### Paso 2: Configurar tu terminal con el mismo dominio

```bash
# 1. Configurar ROS_DOMAIN_ID (CRÍTICO - usar el mismo que el sistema)
export ROS_DOMAIN_ID=5  # Cambiar según el valor obtenido arriba

# 2. Activar el entorno ROS2
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
```

### Paso 3: Verificar conectividad

```bash
# Listar nodos activos (procesos principales)
ros2 node list
```

**Nodos esperados:**
- `/hw_supervisor` ← Siempre presente (control de hardware)
- `/api_gateway` ← Si el servicio obsea3-api-gateway está corriendo
- `/zabbix_bridge` ← Solo si `ZABBIX_ENABLED=true` en .venv

**Ejemplo de salida correcta:**
```
/api_gateway
/hw_supervisor
```

⚠️ **Nota importante**: Los **nodos** son los procesos principales. Para ver los **datos** (ports, rails, env, etc.), usa `ros2 topic list` - ver sección 5 más abajo.

**¿Cuál es la diferencia?**
- **Nodo** = Aplicación que ejecuta (ej: `hw_supervisor`)
- **Topic** = Canal donde se publican datos (ej: `/obsea3/port/1/telemetry`)
- El nodo `hw_supervisor` publica ~30 topics (8 puertos + rails + env + gpio)

**Si NO ves nodos** (lista vacía), verifica que los dominios coincidan:
```bash
echo "Mi ROS_DOMAIN_ID: $ROS_DOMAIN_ID"
grep ROS_DOMAIN_ID /opt/obsea/bin/obsea3.env
# Deben ser el mismo valor (ej: ambos = 5)
```

💡 **Tip:** Para automatizar, añade esto a tu `~/.bashrc`:

```bash
# OBSEA3 - Configuración automática
# Leer ROS_DOMAIN_ID desde .venv (se actualiza dinámicamente según hostname)
if [ -f /opt/obsea/bin/.venv ]; then
    source /opt/obsea/bin/.venv
fi

# Activar ROS2
if [ -f /opt/ros/jazzy/setup.bash ]; then
    source /opt/ros/jazzy/setup.bash
fi
if [ -f /opt/obsea/ros2_ws/install/setup.bash ]; then
    source /opt/obsea/ros2_ws/install/setup.bash
fi
```

⚠️ **IMPORTANTE**: Si cambias el hostname o reinstalar el sistema, **siempre verifica el ROS_DOMAIN_ID** antes de usar ROS2. Dominios diferentes = nodos invisibles.

---

## 1. Ver Telemetría de un Puerto

### Estructura de Topics por Puerto

Cada puerto publica **métricas individuales**:

```bash
# Puerto 1 - Voltaje
ros2 topic echo /obsea3/ports/p1/voltage

# Puerto 1 - Corriente
ros2 topic echo /obsea3/ports/p1/current

# Puerto 1 - Potencia
ros2 topic echo /obsea3/ports/p1/power

# Puerto 1 - Voltaje de retorno
ros2 topic echo /obsea3/ports/p1/vreturn
```

**Ejemplo de salida (voltage):**
```yaml
data: 12.45
---
```

### Telemetría Agregada (todos los puertos)

Para ver **todos los puertos** en un solo topic:

```bash
ros2 topic echo /obsea3/ports/telemetry
```

### Frecuencia de Publicación

```bash
# Ver cada cuánto se publica (debería ser ~5 Hz)
ros2 topic hz /obsea3/ports/p1/voltage
```

### Vía API REST (Recomendado para datos completos)

Para obtener **todos los datos de un puerto** en formato JSON:

```bash
curl http://localhost:8101/ports/1/telemetry
```

**Salida:**
```json
{
  "mode": "12V",
  "voltage": 12.45,
  "current": 0.69,
  "power": 8.59,
  "vreturn": 0.21,
  "link_ok": true,
  "admin_status": 1,
  "oper_status": 1
}
```

---

## 2. Cambiar Modo de un Puerto (ROS2 Service)

### 🚨 ADVERTENCIA DE SEGURIDAD

**Este método BYPASEA toda autenticación de usuario, roles y permisos.**

El servicio ROS2 `/obsea3/set_port_mode` es **interno** y solo valida el token de sistema (no usuarios individuales). Fue diseñado para que la API REST y otros servicios internos puedan comunicarse con el hw_supervisor.

### Arquitectura de Seguridad

```
✅ CORRECTO (Producción):
Usuario → API REST (valida JWT + permisos) → ROS2 Service (token sistema)

⚠️ BYPASS (Solo debugging):
Desarrollador → ROS2 Service directo (token sistema) → SIN validación de permisos
```

### ⚠️ Para Usuarios Finales

**NO uses este método en producción.** Para uso normal, usar la [API REST](API_EJEMPLOS.md#4-controlar-puertos) con autenticación JWT por usuario.

### Cambiar Modo con ROS2 Service (Solo Desarrollo)

```bash
# Obtener token de sistema (requiere sudo - acceso root = acceso total)
SYSTEM_TOKEN=$(sudo grep OBSEA3_TOKEN /opt/obsea/bin/secrets | cut -d'"' -f2)

# Cambiar puerto 1 a 12V (SIN verificar permisos de usuario)
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$SYSTEM_TOKEN'}"

# Cambiar puerto 5 a 48V
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 5, mode: '48V', token: '$SYSTEM_TOKEN'}"

# Apagar puerto 1
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: 'OFF', token: '$SYSTEM_TOKEN'}"
```

**Respuesta exitosa:**
```yaml
success: true
message: "Port 1 set to 12V"
```

### ⚠️ Por Qué NO Usa Autenticación por Usuario

El servicio ROS2 es una **capa interna** que:
- Solo valida que el token coincida con `OBSEA3_TOKEN` (sistema)
- NO verifica usuarios, roles, ni permisos en puertos
- Confía en que el llamador (API REST) ya validó permisos
- Permite comunicación entre servicios internos (API ↔ hw_supervisor)

**Casos de uso legítimos:**
- Debugging durante desarrollo
- Testing automático de integración
- Scripts de mantenimiento con acceso root
- Recuperación de emergencia

**❌ NO usar para:**
- Operación normal del sistema
- Usuarios sin privilegios root
- Producción o acceso remoto

---

## 2b. Cambiar Modo con Autenticación de Usuario (ROS2 + JWT)

### ✅ Método Recomendado para Usuarios

Desde la versión actual, `hw_supervisor` soporta **autenticación dual**:
1. **Token de sistema** (OBSEA3_TOKEN) → solo debugging/admin
2. **JWT de usuario** (username + token) → producción, con roles y permisos

### Cómo Funciona

El servicio `/obsea3/set_port_mode` ahora acepta:
- `username`: Nombre del usuario
- `token`: JWT token del usuario (largo, ~200 caracteres)
- `lease_id`: ID del lease activo (opcional)

El `hw_supervisor` validará:
- ✅ JWT válido (firma, expiración)
- ✅ Usuario tiene rol adecuado (admin/operator)
- ✅ Usuario tiene permiso en ese puerto específico
- ✅ Lease válido (si se provee)

### Paso 1: Obtener JWT Token

```bash
# Login con usuario/password → obtener JWT
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" \
  -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

echo "JWT obtenido: ${JWT:0:50}..."
```

### Paso 2: Adquirir Lease (Opcional pero Recomendado)

```bash
# Adquirir lease para puerto 1
LEASE=$(curl -s -X POST "http://localhost:8101/ports/1/lease/acquire" \
  -H "Authorization: Bearer $JWT" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['lease_id'])")

echo "Lease adquirido: $LEASE"
```

### Paso 3: Cambiar Modo con Autenticación

```bash
# Cambiar puerto 1 a 12V con JWT + lease (usuario con permisos)
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$JWT', username: '<usuario>', lease_id: '$LEASE'}"
```

**Respuesta exitosa:**
```yaml
success: true
message: "Port 1 set to 12V"
```

### Paso 4: Liberar Lease (Buena Práctica)

```bash
# Liberar lease cuando termines
curl -s -X POST "http://localhost:8101/ports/1/lease/release" \
  -H "Authorization: Bearer $JWT" \
  -d "lease_id=$LEASE"
```

### Ejemplos Completos

#### Ejemplo 1: Usuario con Permisos (Éxito)

```bash
# Usuario 'test' (viewer, sin permisos de puerto)
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# Intento de cambiar puerto 1 a 12V (fallará por falta de permisos)
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$JWT', username: 'test', lease_id: ''}"

# Resultado: ✅ success: true
```

#### Ejemplo 2: Usuario sin Permisos (Fallo)

```bash
# Usuario 'test' (viewer, solo lectura)
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$JWT', username: 'test', lease_id: ''}"

# Resultado: ❌ success: false, message: "User 'test' not authorized for port 1"
```

#### Ejemplo 3: Con Lease (Control Exclusivo)

```bash
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" -d "username=operador&password=<password>" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

LEASE=$(curl -s -X POST "http://localhost:8101/ports/1/lease/acquire" \
  -H "Authorization: Bearer $JWT" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['lease_id'])")

# Ahora solo operador puede controlar puerto 1 por 60s
ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '48V', token: '$JWT', username: 'operador', lease_id: '$LEASE'}"

# Otro usuario intentando sin lease → falla con "Port locked by operador"
```

### Script Bash Completo

```bash
#!/bin/bash
# control_port.sh - Cambiar modo de puerto con autenticación

PORT=${1:-1}
MODE=${2:-12V}
USERNAME=${3:-operador}
PASSWORD=${4:-<password>}

# Configurar ROS_DOMAIN_ID
source /opt/obsea/bin/.venv

# Login
echo "Autenticando como $USERNAME..."
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" \
  -d "username=$USERNAME&password=$PASSWORD" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])" 2>/dev/null)

if [ -z "$JWT" ]; then
  echo "❌ Error: Login fallido"
  exit 1
fi

echo "✅ JWT obtenido: ${JWT:0:30}..."

# Adquirir lease
echo "Adquiriendo lease para puerto $PORT..."
LEASE=$(curl -s -X POST "http://localhost:8101/ports/$PORT/lease/acquire" \
  -H "Authorization: Bearer $JWT" | \
  python3 -c "import sys, json; print(json.load(sys.stdin).get('lease_id', ''))" 2>/dev/null)

if [ -n "$LEASE" ]; then
  echo "✅ Lease adquirido: $LEASE"
else
  echo "⚠️  No se pudo adquirir lease (puede estar en uso)"
fi

# Cambiar modo
echo "Cambiando puerto $PORT a $MODE..."
RESULT=$(ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
  "{port: $PORT, mode: '$MODE', token: '$JWT', username: '$USERNAME', lease_id: '$LEASE'}")

if echo "$RESULT" | grep -q "success: true"; then
  echo "✅ Puerto $PORT → $MODE"
else
  echo "❌ Error: $(echo "$RESULT" | grep message)"
fi

# Liberar lease si se obtuvo
if [ -n "$LEASE" ]; then
  echo "Liberando lease..."
  curl -s -X POST "http://localhost:8101/ports/$PORT/lease/release" \
    -H "Authorization: Bearer $JWT" \
    -d "lease_id=$LEASE" > /dev/null
  echo "✅ Lease liberado"
fi
```

**Uso:**
```bash
chmod +x control_port.sh
./control_port.sh 1 12V operador <password>
./control_port.sh 2 48V operador <password>
```

### Comparación: API REST vs ROS2 + JWT

| Característica | API REST | ROS2 + JWT | ROS2 + Token Sistema |
|----------------|----------|------------|---------------------|
| Autenticación usuario | ✅ | ✅ | ❌ |
| Validación de roles | ✅ | ✅ | ❌ |
| Permisos por puerto | ✅ | ✅ | ❌ |
| Leases (exclusividad) | ✅ | ✅ | ❌ |
| Audit logging | ✅ | ⚠️ Parcial | ❌ |
| Uso recomendado | Usuarios finales | Scripts avanzados | Solo debugging |

### Troubleshooting

#### Error: "Invalid JWT token"
```bash
# Verificar que el JWT no haya expirado (15min TTL)
# Hacer login de nuevo
JWT=$(curl -s -X POST "http://localhost:8101/auth/login" -d "username=operador&password=<password>" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")
```

#### Error: "User not authorized for port N"
```bash
# Verificar permisos del usuario
curl -s "http://localhost:8101/auth/debug_me" -H "Authorization: Bearer $JWT"

# Añadir permiso (como admin)
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py grant-port operador 1
```

#### Error: "Port locked by user_id=X"
```bash
# Ver lease actual
curl -s "http://localhost:8101/ports/1/lease"

# Esperar a que expire (60s) o liberar como admin
sudo sqlite3 /opt/obsea/data/auth.db "DELETE FROM leases WHERE port=1;"
sudo systemctl restart obsea3-hw-supervisor
```

---

## 3. Ver Telemetría de los Buses (Rails)

Cada bus (rail) publica **métricas individuales**:

```bash
# Bus 12V - Voltaje
ros2 topic echo /obsea3/rails/v12/voltage

# Bus 12V - Corriente
ros2 topic echo /obsea3/rails/v12/current

# Bus 12V - Potencia
ros2 topic echo /obsea3/rails/v12/power

# Bus 48V
ros2 topic echo /obsea3/rails/v48/voltage
ros2 topic echo /obsea3/rails/v48/current
ros2 topic echo /obsea3/rails/v48/power

# Bus 5V
ros2 topic echo /obsea3/rails/v5/voltage
ros2 topic echo /obsea3/rails/v5/current
ros2 topic echo /obsea3/rails/v5/power
```

**Ejemplo de salida:**
```yaml
data: 12.34
---
```

---

## 4. Ver Estado de GPIO

Ver el estado actual de las máscaras de control:

```bash
ros2 topic echo /obsea3/gpio_state
```

**Ejemplo de salida:**
```yaml
data: |
  ports:
    1: '12V'
    2: 'OFF'
    3: 'OFF'
    4: 'OFF'
    5: '48V'
    6: '48V'
    7: 'OFF'
    8: 'OFF'
  enable_mask: 33
  select_mask: 48
---
```

---

## 5. Listar Todos los Topics

Ver todos los tópicos disponibles:

```bash
ros2 topic list
```

**Salida típica (estructura real):**
```
/obsea3/env/humidity
/obsea3/env/pressure
/obsea3/env/temperature
/obsea3/limits/warnings
/obsea3/ports/gpio_state
/obsea3/ports/p1/voltage
/obsea3/ports/p1/current
/obsea3/ports/p1/power
/obsea3/ports/p1/vreturn
... (p2 a p8 con mismas métricas)
/obsea3/ports/telemetry
/obsea3/rails/v12/voltage
/obsea3/rails/v12/current
/obsea3/rails/v12/power
/obsea3/rails/v48/voltage
/obsea3/rails/v48/current
/obsea3/rails/v48/power
/obsea3/rails/v5/voltage
/obsea3/rails/v5/current
/obsea3/rails/v5/power
```

---

## 6. Ver Información de un Topic

Ver detalles de un tópico (tipo de mensaje, publicadores, suscriptores):

```bash
ros2 topic info /obsea3/ports/p1/voltage
```

**Salida:**
```
Type: obsea3_interfaces/msg/PortTelemetry
Publisher count: 1
Subscription count: 0
```

---

## 7. Ver Definición de un Mensaje

Ver la estructura completa de un tipo de mensaje:

```bash
ros2 interface show obsea3_interfaces/msg/PortTelemetry
```

**Salida:**
```
float32 voltage
float32 current
float32 power
float32 vreturn
bool link_ok
int32 ifindex
int32 admin_status
int32 oper_status
```

---

## 8. Grabar Datos (Bag)

Grabar telemetría para análisis posterior:

```bash
# Grabar todos los puertos durante 60 segundos
ros2 bag record -o mi_sesion /obsea3/port/1/telemetry /obsea3/port/2/telemetry --duration 60

# Grabar TODOS los topics del namespace obsea3
ros2 bag record -o sesion_completa -a /obsea3
```

**Reproducir grabación:**
```bash
ros2 bag play mi_sesion
```

---

## 9. Monitorear Varios Puertos Simultáneamente

Usando tmux o múltiples terminales:

```bash
# Terminal 1 - Puerto 1
ros2 topic echo /obsea3/port/1/telemetry

# Terminal 2 - Puerto 5
ros2 topic echo /obsea3/port/5/telemetry

# Terminal 3 - Bus 12V
ros2 topic echo /obsea3/rails/v12
```

---

## Troubleshooting

### Error: "Could not determine the type for the passed topic"

**Causa:** El nodo hw_supervisor no está publicando el topic todavía.

**Solución:**
```bash
# Verificar que el nodo está corriendo
ros2 node list

# Si no aparece, iniciar el sistema
cd /opt/obsea
bash bin/50_run_all.sh
```

### Error: "service call timeout"

**Causa:** El servicio no está disponible o hay problema de ROS_DOMAIN_ID.

**Solución:**
```bash
# 1. Verificar que el servicio existe
ros2 service list | grep set_port_mode

# 2. Verificar ROS_DOMAIN_ID
echo $ROS_DOMAIN_ID  # Debe ser 0

# 3. Si está mal, configurar correctamente
export ROS_DOMAIN_ID=0
```

### Error: "Invalid token"

**Causa:** El token no coincide con el configurado en el sistema.

**Solución:**
```bash
# Ver el token correcto
sudo cat /opt/obsea/bin/secrets | grep OBSEA3_TOKEN

# Copiar el valor (sin comillas ni export) y usarlo en el comando
```

---

## Scripts de Ejemplo

### Script para monitorear corriente de todos los puertos

Crear `monitor_currents.sh`:

```bash
#!/bin/bash
export ROS_DOMAIN_ID=0
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash

echo "=== Monitoreando corriente de todos los puertos ==="
for port in {1..8}; do
    echo -n "Puerto $port: "
    ros2 topic echo --once /obsea3/port/$port/telemetry | grep current | cut -d' ' -f2
done
```

Ejecutar:
```bash
chmod +x monitor_currents.sh
./monitor_currents.sh
```

### Script para apagar todos los puertos

Crear `shutdown_all.sh`:

```bash
#!/bin/bash
export ROS_DOMAIN_ID=0
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash

# IMPORTANTE: Sustituir YOUR_TOKEN
TOKEN="YOUR_TOKEN_AQUI"

echo "=== Apagando todos los puertos ==="
for port in {1..8}; do
    echo "Apagando puerto $port..."
    ros2 service call /obsea3/set_port_mode obsea3_interfaces/srv/SetPortMode \
        "{port: $port, mode: 'OFF', token: '$TOKEN'}"
done
echo "=== Todos los puertos apagados ==="
```

---

## Recursos Adicionales

- **Documentación completa:** `/home/carla/obsea/docs/`
- **Logs del sistema:** `/opt/obsea/log/current/`
- **Configuración:** `/opt/obsea/data/*.yaml`
- **Web UI:** `http://localhost:8180` (usuario/password según configuración)

---

## Contacto

Para dudas o problemas, revisar:
1. Logs: `tail -f /opt/obsea/log/current/hw_supervisor.log`
2. Estado del sistema: `bash /opt/obsea/bin/remote_diagnostics.sh`
3. Alarmas activas: `bash /opt/obsea/bin/more/check_alarms.sh`
