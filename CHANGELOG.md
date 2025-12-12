# Changelog

Todos los cambios notables del proyecto OBSEA3 se documentan en este archivo.

El formato está basado en [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
y este proyecto adhiere a [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added
- **🔐 Autenticación de usuario en ROS2 services** (diciembre 2025)
  - `hw_supervisor` ahora valida JWT tokens de usuarios (no solo token de sistema)
  - Soporte dual: JWT con roles/permisos O token de sistema (retrocompatibilidad)
  - Validación de permisos por puerto en capa ROS2 (antes solo en API)
  - Sistema de leases integrado en hw_supervisor (control exclusivo)
  - Nuevos módulos: `auth_validator.py`, `lease_validator.py`
  - Interface `SetPortMode.srv` extendido: añadidos campos `username` y `lease_id`
  - Documentación completa en `docs/ROS2_EJEMPLOS.md` con ejemplos JWT
- Sistema de monitoreo Zabbix integrado con systemd service
- Soporte para python3-zabbix-utils (compatible con ARM64)
- 45 métricas monitoreadas: ports (32), rails (6), environment (3), limits (1)
- Variables dinámicas en servicios systemd (ROS_DOMAIN_ID, DUMMY, ZABBIX_ENABLED)

### Fixed
- **auth_validator.py**: Columnas de base de datos corregidas (roles.name, user_ports.port, users.is_active)
- **hw_supervisor**: Conversión de tipos user_id (str → int) para validación de leases
- Parámetro `dummy` ahora se pasa correctamente a `hw_supervisor_node` desde systemd
- Valores dummy para rails (v12/v48) y environment (temp/humidity/pressure) publicándose
- ROS_DOMAIN_ID hardcodeado → ahora lee de `obsea3.env` dinámicamente
- api-gateway ahora exporta ROS_DOMAIN_ID para clase RosMirror
- Zabbix service verifica ZABBIX_ENABLED antes de arrancar

### Changed
- **Arquitectura de seguridad**: Autenticación dual en API y ROS2 (antes solo API)
- **API gateway**: Ahora pasa JWT + username + lease_id a hw_supervisor (no solo token sistema)
- Logging mejorado en zabbix_bridge: ✓ Enviado / ✗ Rechazado con detalles
- Hostname Zabbix corregido: obsea3-02 → obsea3-03

### Security
- **BREAKING (seguridad)**: ROS2 service `/obsea3/set_port_mode` ahora valida permisos de usuario
  - Scripts que usan token de sistema siguen funcionando (retrocompatibilidad)
  - Uso con JWT requiere rol adecuado (admin/operator) y permiso en puerto específico
  - Leases previenen conflictos de acceso concurrente entre usuarios

---

## [2025-12-11] - Instalación Inicial y Zabbix

### Added
- Script de emergencia para permisos: `bin/fix_permissions_emergency.sh`
- Verificación automática de grupo `obsea_admins` en `00_install_all.sh`
- Documentación completa de troubleshooting en ARCHITECTURE.md
- Sistema Zabbix con 45 métricas monitoreadas

### Fixed
- **CRÍTICO**: Permisos de grupo - usuario debe estar en `obsea_admins`
  - Error misleading "No space left on device" → causa real: permisos `/opt/obsea/bin/.venv`
  - `00_install_all.sh` usa `sg obsea_admins` para build con permisos correctos
  - `30_build_ros2_ws.sh` limpia build/install owned by root automáticamente
- node_modules corrompido en webtest
  - Reinstalación completa: `npm install` en Step 4.6
  - 42,789 archivos eliminados del repositorio git
  - rsync excluye node_modules/dist/.vite
- `.gitignore` corregido: `package-lock.json` SÍ debe trackearse
- `package.json` locks explícitos para dependencias críticas:
  - vite: "5.4.0" (no "^5.4.0" para evitar breaking changes)
  - vitest: "^2.0.0" pinned
- ROS_DOMAIN_ID faltante en `~/.bashrc`
  - `00_interactive_setup.sh` lo añade automáticamente
  - Previene errores "Node not discovered" en ROS2
- Instalación fallaba si ya existía `/opt/obsea/`
  - Ahora verifica existencia antes de crear
- Systemd service Zabbix con ROS_DOMAIN_ID correcto
- Zabbix usando python3-zabbix-utils en lugar de binario (ARM64 compatible)
- Hostname Zabbix corregido para coincidir con servidor

### Changed
- `00_install_all.sh` ahora verifica grupo en Step 2.5
- Build usa usuario correcto (no root) vía `sg obsea_admins`
- Logs organizados por fecha: `log/YYYY/MM/DD/*.log` con symlink `log/current/`

---

## [2025-12-10] - Configuración Web y Base de Datos

### Added
- Script `bin/more/list_api_users.sh` para listar usuarios API
- Verificación de Node.js 18+ en scripts de instalación

### Fixed
- **Vite "not found"**:
  - Downgrade vite 7.x → 5.4.0 (compatible con Node.js 18+)
  - Evita error `crypto.hash is not a function`
- **Configuración incorrecta de puertos**:
  - Puerto y host movidos a `vite.config.js` (centralizados)
  - Scripts simplificados: `npm run dev` sin argumentos CLI
- **IP y puerto hardcodeados en frontend**:
  - `App.jsx`: Eliminadas IPs hardcodeadas, ahora usa variable de entorno
  - Debe coincidir con `UVICORN_PORT` del `.venv`
- **Pérdida de usuarios al reinstalar**:
  - Backup automático de `/opt/obsea/data/*.db` antes de reinstalar
  - Restauración automática post-instalación
  - Protección contra borrado accidental de datos

### Changed
- Base de datos SQLite movida de ROS workspace a `/opt/obsea/data/`
- Instrucciones de instalación mejoradas en README

---

## [2025-12-09] - Sistema de Alarmas

### Added
- Sistema de deduplicación de alarmas con delay de recuperación (10s)
- Tracking de estado de alarmas: `_port_limit_state` y `_bus_limit_state`
- Logging mejorado: mensajes de recuperación cuando alarmas se resuelven
- API endpoint: `GET /telemetry/alarms` para consultar alarmas activas

### Fixed
- **Spam de alarmas**:
  - Alarmas duplicadas cada 5 segundos para misma condición
  - Recovery delay de 10s antes de resetear estado
  - Previene spam por fluctuaciones cercanas al threshold
- Logging duplicado:
  - Overcurrent, undervolt, overvolt solo se loggean una vez
  - Mensaje de recuperación INFO cuando condición se estabiliza
  - Flags `*_reported` previenen duplicados

### Changed
- Alarmas requieren 10s de estabilidad antes de considerarse resueltas
- Estados de alarma persisten en diccionarios en memoria

---

## Documentación Histórica

Para detalles completos de debugging y problemas específicos, ver:
- [docs/archive/ALARM_DEDUP_FIX.md](docs/archive/ALARM_DEDUP_FIX.md)
- [docs/archive/CHANGELOG_2025-12-10.md](docs/archive/CHANGELOG_2025-12-10.md)
- [docs/archive/CHANGELOG_2025-12-11.md](docs/archive/CHANGELOG_2025-12-11.md)
