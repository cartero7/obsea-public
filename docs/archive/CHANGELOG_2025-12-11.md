# CHANGELOG - 11 Diciembre 2025

## Problemas resueltos durante la instalación inicial

### 1. Permisos de grupo (CRÍTICO)
**Problema**: Usuario no podía leer `/opt/obsea/bin/.venv` → error "No space left on device" (misleading)

**Solución aplicada**:
- Usuario debe estar en grupo `obsea_admins`: `sudo usermod -a -G obsea_admins $USER`
- Script `00_install_all.sh` ahora usa `sg obsea_admins` para ejecutar build con permisos correctos
- Script de emergencia creado: `bin/fix_permissions_emergency.sh`

**Archivos modificados**:
- `bin/00_install_all.sh` - Step 2.5 (verificación grupo), Step 5 (sg obsea_admins)
- `bin/30_build_ros2_ws.sh` - Auto-cleanup de build/install owned by root
- `.github/ARCHITECTURE.md` - Documentado troubleshooting

### 2. node_modules corrompido
**Problema**: Vite no encontrado, symlinks rotos en `webtest/node_modules/`

**Solución aplicada**:
- Reinstalación completa: `cd /opt/obsea/webtest && npm install`
- Eliminados 42,789 archivos de node_modules del repositorio git
- Integrado en `00_install_all.sh` Step 4.6

**Archivos modificados**:
- `bin/00_install_all.sh` - Step 4.6 (npm install)
- `bin/02_host_setup.sh` - rsync exclude node_modules/dist/.vite
- `.gitignore` - Corregido (package-lock.json SÍ debe trackearse)

### 3. Web frontend no arrancaba
**Problema**: `npm run dev` no encontraba vite en PATH

**Solución aplicada**:
- Cambio de `npm run dev` → `npx vite` con PATH explícito
- `export PATH="/opt/obsea/webtest/node_modules/.bin:$PATH"`

**Archivos modificados**:
- `bin/50_run_all.sh` - Línea 177
- `bin/obsea3_web_run.sh` - Línea similar

### 4. Base de datos sin inicializar
**Problema**: SQLite error "no such table: users"

**Solución aplicada**:
- Migrations integradas en instalación automática
- `00_install_all.sh` Step 4.5 llama a `20_db_migrate_obsea_api.sh` y `21_db_migrate_auth.sh`

**Archivos modificados**:
- `bin/00_install_all.sh` - Step 4.5
- `bin/21_db_migrate_auth.sh` - Creación de tablas + migraciones

### 5. Schema incompleto en auth.db
**Problema**: Columnas faltantes causaban errores en `obsea_user_admin.py`
- `users.is_active` - ❌ faltaba
- `users.updated_at` - ❌ faltaba  
- `roles.created_at` - ❌ faltaba
- `user_roles.created_at` - ❌ faltaba

**Solución aplicada**:
- Agregadas todas las columnas faltantes
- `21_db_migrate_auth.sh` ahora detecta y migra schemas antiguos

**Archivos modificados**:
- `bin/21_db_migrate_auth.sh` - Section 3 (migración columnas)

### 6. Endpoint /telemetry/alarms faltante (404)
**Problema**: Web llamaba a `/telemetry/alarms` pero endpoint no existía

**Solución aplicada**:
- Función `get_active_alarms()` en `actions_log.py`
- Endpoint `GET /telemetry/alarms` en `main.py`

**Archivos modificados**:
- `ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/actions_log.py` - Función nueva
- `ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/main.py` - Endpoint nuevo

### 7. .gitignore no funcionaba
**Problema**: 42,789 archivos de node_modules trackeados en git

**Solución aplicada**:
- `git rm -r --cached webtest/node_modules/ webtest/.vite/`
- Limpieza completa del repositorio

**Archivos modificados**:
- `.gitignore` - Comentario sobre package-lock.json

---

## Verificación post-instalación

**Comandos para verificar que todo funciona**:

```bash
# 1. Servicios corriendo
ps aux | grep -E "(hw_supervisor|uvicorn|vite)" | grep -v grep

# 2. Base de datos OK
sqlite3 /opt/obsea/data/auth.db ".schema users"
sqlite3 /opt/obsea/data/auth.db "SELECT * FROM roles;"

# 3. Admin de usuarios funciona
python3 /opt/obsea/bin/more/obsea_user_admin.py

# 4. API responde
curl http://localhost:8101/telemetry/alarms

# 5. Web accesible
curl http://localhost:8080
```

---

## Usuario por defecto creado

```
Username: test
Password: test123
Role: viewer (solo lectura)
```

Para crear admin: `python3 bin/more/obsea_user_admin.py`

---

## Commits realizados

1. **3ffa9bf1** - fix: Correcciones instalación y alarms endpoint (10 archivos)
2. **b66320b3** - chore: Eliminar node_modules del repositorio (42,792 archivos)

---

## Próxima instalación limpia

Ejecutar simplemente:
```bash
bash bin/00_install_all.sh
```

Esto ahora incluye TODAS las correcciones aplicadas hoy.

---

### 8. Systemd services para persistencia
**Problema**: Servicios corrían en foreground y se paraban al cerrar terminal. ROS_DOMAIN_ID=5 no se propagaba correctamente.

**Solución aplicada**:
- Creados 3 servicios systemd:
  - `obsea3-hw-supervisor.service` - Hardware supervisor con ROS2
  - `obsea3-api-gateway.service` - FastAPI + ROS2 bridge
  - `obsea3-web.service` - Vite dev server
- Script de instalación: `bin/60_install_systemd_services.sh`
- Environment file limpio: `bin/obsea3.env` (KEY=VALUE, sin bash syntax)
- **CRÍTICO**: `export ROS_DOMAIN_ID=5` ANTES de source en ExecStart

**Archivos creados**:
- `systemd/obsea3-hw-supervisor.service`
- `systemd/obsea3-api-gateway.service`
- `systemd/obsea3-web.service`
- `bin/60_install_systemd_services.sh`
- `bin/obsea3.env` (template, copiado a /opt/obsea/bin/)

**Características**:
- Auto-arranque en boot (`WantedBy=multi-user.target`)
- Auto-reinicio en fallo (`Restart=on-failure`)
- Logs persistentes en `/opt/obsea/log/current/`
- Control con: `sudo systemctl {start|stop|restart|status} obsea3-*`

### 9. Configuración backplane.yaml incompleta
**Problema**: Solo 3 puertos configurados en lugar de 8. Telemetría solo mostraba puertos 1-3.

**Solución aplicada**:
- Restaurados puertos 4-8 con direcciones I2C correctas (LTC2945 0x6B-0x67)
- Añadido `ifindex_map` completo para SNMP (puertos 1-8 + Raspberry 9-10 + WAN 15-16)
- Sincronizado en 3 ubicaciones: `/opt/obsea/config/`, source en `ros2_ws/src/`, dev en `~/obsea/`

**Archivos modificados**:
- `/opt/obsea/config/backplane.yaml`
- `/opt/obsea/ros2_ws/src/obsea3_core/config/backplane.yaml`
- `/home/carla/obsea/ros2_ws/src/obsea3_core/config/backplane.yaml`

**Resultado**: Telemetría funcional para 8 puertos completos
