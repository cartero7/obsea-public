# OBSEA3 - Ejemplos de Uso de la API REST

Este documento muestra cómo interactuar con el sistema OBSEA3 usando la **API REST** (recomendado para usuarios finales).

> 📘 **¿Buscas comandos ROS2?** Ver [ROS2_EJEMPLOS.md](ROS2_EJEMPLOS.md) (para desarrollo/debugging)

---

## Requisitos Previos

- Sistema OBSEA3 corriendo (servicios activos)
- Usuario creado con permisos adecuados (ver [USUARIOS_Y_DATOS.md](USUARIOS_Y_DATOS.md))
- `curl` instalado (viene por defecto en Ubuntu)

---

## 1. Health Check

Verificar que la API está funcionando:

```bash
curl http://localhost:8101/health
```

**Respuesta esperada:**
```json
{
  "ok": true,
  "node": "api_gateway"
}
```

---

## 2. Autenticación

### Login y Obtener Token JWT

```bash
# Login con usuario (ej: test/test123)
TOKEN=$(curl -s -X POST http://localhost:8101/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

echo "JWT Token: $TOKEN"
```

**Respuesta esperada:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

### Verificar Información del Usuario

```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8101/me/ports
```

**Respuesta:**
```json
{
  "username": "test",
  "role": "viewer",
  "ports": []
}
```

---

## 3. Ver Telemetría

### Telemetría de un Puerto Específico

```bash
# Ver telemetría del puerto 1
curl http://localhost:8101/ports/1/telemetry | python3 -m json.tool
```

**Respuesta:**
```json
{
  "mode": "12V",
  "voltage": 12.45,
  "current": 0.69,
  "power": 8.59,
  "vreturn": 0.21,
  "stamp": {
    "sec": 1765520204,
    "nanosec": 680700238
  },
  "ifindex": 1,
  "admin_status": 1,
  "oper_status": 1,
  "link_ok": true
}
```

### Telemetría de Todos los Puertos

```bash
curl http://localhost:8101/telemetry | python3 -m json.tool
```

### Estado de GPIO (Máscaras de Control)

```bash
curl http://localhost:8101/gpio_state | python3 -m json.tool
```

### Alarmas Activas

```bash
curl http://localhost:8101/telemetry/alarms | python3 -m json.tool
```

---

## 4. Controlar Puertos

### ⚠️ Requisitos de Permisos

Para cambiar el modo de un puerto, el usuario debe tener:
- **Rol**: `operator` o `admin` (NO `viewer`)
- **Permisos asignados** en ese puerto específico

Verificar permisos:
```bash
curl -H "Authorization: Bearer $TOKEN" http://localhost:8101/me/ports
```

### Cambiar Modo del Puerto

```bash
# Cambiar puerto 1 a 12V
curl -X POST http://localhost:8101/ports/1/mode \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"12V"}' | python3 -m json.tool

# Cambiar puerto 5 a 48V
curl -X POST http://localhost:8101/ports/5/mode \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"48V"}' | python3 -m json.tool

# Apagar puerto 1
curl -X POST http://localhost:8101/ports/1/mode \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"OFF"}' | python3 -m json.tool
```

**Respuesta exitosa:**
```json
{
  "success": true,
  "port": 1,
  "mode": "12V"
}
```

**Error si no tienes permisos:**
```json
{
  "detail": "No tienes permiso para accionar este puerto"
}
```

---

## 5. Sistema de Leases (Control Exclusivo)

### Adquirir Lease (Control Exclusivo de un Puerto)

```bash
# Adquirir control exclusivo del puerto 1 por 60 segundos
curl -X POST http://localhost:8101/ports/1/lease/acquire \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

**Respuesta:**
```json
{
  "port": 1,
  "owner": "test",
  "expires_at": "2025-12-12T07:30:00",
  "token": "lease_abc123..."
}
```

### Verificar Estado del Lease

```bash
curl http://localhost:8101/ports/1/lease | python3 -m json.tool
```

### Renovar Lease (Heartbeat)

```bash
curl -X POST http://localhost:8101/lock/port_1/heartbeat \
  -H "Authorization: Bearer $TOKEN"
```

### Liberar Lease

```bash
curl -X POST http://localhost:8101/ports/1/lease/release \
  -H "Authorization: Bearer $TOKEN"
```

---

## 6. Configuración

### Ver Configuración de Instrumentos

```bash
curl http://localhost:8101/config/instruments | python3 -m json.tool
```

### Ver Configuración de Límites

```bash
curl http://localhost:8101/config/limits | python3 -m json.tool
```

### Modificar Configuración (Solo Admin)

```bash
# Descargar configuración actual
curl http://localhost:8101/config/instruments > instruments.yaml

# Editar archivo...
nano instruments.yaml

# Subir configuración modificada
curl -X PUT http://localhost:8101/config/instruments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: text/plain" \
  --data-binary @instruments.yaml
```

---

## 7. Logs y Auditoría

### Últimas Acciones en un Puerto

```bash
curl http://localhost:8101/ports/1/last_action | python3 -m json.tool
```

### Acciones en Todos los Puertos

```bash
curl http://localhost:8101/ports/actions | python3 -m json.tool
```

### Eventos del Sistema

```bash
curl http://localhost:8101/logs/system | python3 -m json.tool
```

---

## 8. Documentación Interactiva

La API incluye documentación interactiva generada automáticamente:

```bash
# Swagger UI (interfaz gráfica)
http://localhost:8101/docs

# ReDoc (documentación alternativa)
http://localhost:8101/redoc

# Schema OpenAPI en JSON
http://localhost:8101/openapi.json
```

---

## 9. Ejemplos Completos

### Script Bash: Monitorear Puerto y Cambiar Modo

```bash
#!/bin/bash

# Login
TOKEN=$(curl -s -X POST http://localhost:8101/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=operator&password=secret123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# Ver telemetría actual
echo "=== Telemetría Puerto 1 ==="
curl -s http://localhost:8101/ports/1/telemetry | python3 -m json.tool

# Cambiar a 12V
echo -e "\n=== Cambiar a 12V ==="
curl -s -X POST http://localhost:8101/ports/1/mode \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"12V"}' | python3 -m json.tool

# Esperar 5 segundos
sleep 5

# Ver telemetría después del cambio
echo -e "\n=== Telemetría después del cambio ==="
curl -s http://localhost:8101/ports/1/telemetry | python3 -m json.tool
```

### Script Python: Control Automatizado

```python
#!/usr/bin/env python3
import requests
import time

# Configuración
API_URL = "http://localhost:8101"
USERNAME = "operator"
PASSWORD = "secret123"

# Login
response = requests.post(
    f"{API_URL}/auth/login",
    data={"username": USERNAME, "password": PASSWORD}
)
token = response.json()["access_token"]
headers = {"Authorization": f"Bearer {token}"}

# Ver telemetría
telemetry = requests.get(f"{API_URL}/ports/1/telemetry").json()
print(f"Puerto 1 - Modo: {telemetry['mode']}, Voltaje: {telemetry['voltage']}V")

# Cambiar modo
response = requests.post(
    f"{API_URL}/ports/1/mode",
    headers=headers,
    json={"mode": "12V"}
)
print(f"Cambio: {response.json()}")

# Monitorear telemetría cada 5 segundos
for i in range(10):
    time.sleep(5)
    telemetry = requests.get(f"{API_URL}/ports/1/telemetry").json()
    print(f"[{i+1}] Corriente: {telemetry['current']}A, Potencia: {telemetry['power']}W")
```

---

## Troubleshooting

### Error: "Unauthorized" o "Invalid credentials"

- Verificar usuario existe: `sudo bash /opt/obsea/bin/03_create_users.sh --list-users`
- Verificar password correcto
- Verificar Content-Type es `application/x-www-form-urlencoded`

### Error: "No tienes permiso para accionar este puerto"

- Usuario debe tener rol `operator` o `admin` (NO `viewer`)
- Usuario debe tener permisos en ese puerto:
  ```bash
  sudo python3 /opt/obsea/bin/more/obsea_user_admin.py grant-port usuario 1
  ```

### Error: "Connection refused"

- Verificar servicio API corriendo:
  ```bash
  systemctl status obsea3-api-gateway
  curl http://localhost:8101/health
  ```

### Token JWT Expirado

Los tokens JWT expiran después de 15 minutos (por defecto). Volver a hacer login:
```bash
TOKEN=$(curl -s -X POST http://localhost:8101/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")
```

---

## Documentos Relacionados

- [ROS2_EJEMPLOS.md](ROS2_EJEMPLOS.md) - Comandos ROS2 para desarrollo/debugging
- [USUARIOS_Y_DATOS.md](USUARIOS_Y_DATOS.md) - Gestión de usuarios y permisos
- [../DEPLOYMENT_GUIDE.md](../DEPLOYMENT_GUIDE.md) - Guía de instalación
