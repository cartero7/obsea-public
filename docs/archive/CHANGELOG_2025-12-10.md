# Resumen de Cambios - Diciembre 10, 2025

## 🎯 Problemas Resueltos

### 1. Error "vite: not found"
**Causa:** Vite 7.x requiere Node.js 20+, pero el sistema tiene Node.js 18.x

**Solución:**
- Downgrade de vite a versión 5.4.0 en `package.json`
- Compatible con Node.js 18.x
- Evita error `crypto.hash is not a function`

### 2. Configuración incorrecta de puertos
**Causa:** Parámetros de vite pasados incorrectamente por línea de comandos

**Solución:**
- Configuración centralizada en `vite.config.js`
- Puerto 8080 definido en archivo de config
- Host 0.0.0.0 para acceso desde red
- Scripts simplificados (`npm run dev` sin argumentos)

### 3. IP y puerto hardcodeados incorrectos en frontend
**Causa:** `App.jsx` tenía IP 192.168.1.152:8001 (incorrecta)

**Solución:**
- Corregida IP a 192.168.1.151
- Puerto corregido a 8101 (el correcto para API)
- Debe coincidir con `UVICORN_PORT` del `.venv`

### 4. Pérdida de usuarios al reinstalar
**Causa:** No había backups automáticos ni documentación

**Solución:**
- Documentación completa en `docs/USUARIOS_Y_DATOS.md`
- Script de post-instalación `bin/99_post_install.sh`
- Guía de backups y restauración
- Checklist pre-migración

## 📝 Archivos Modificados

### Commits realizados:

#### Commit 1: Fix configuración de puertos y vite
```
3bb1e142 - Fix: Configuración correcta de puertos y vite para web
```

**Archivos:**
- `bin/50_run_all.sh` - Simplificado npm run dev
- `bin/obsea3_web_run.sh` - Cambiado npx vite a npm run dev
- `webtest/vite.config.js` - Agregada config de server
- `webtest/src/views/App.jsx` - IP y puerto corregidos
- `docs/USUARIOS_Y_DATOS.md` - Nueva documentación

#### Commit 2: Documentación y post-instalación
```
4da9f115 - Add: Documentación y scripts de post-instalación
```

**Archivos:**
- `INSTALL_QUICKSTART.md` - Guía rápida instalación
- `bin/99_post_install.sh` - Script post-instalación
- `webtest/package.json` - Vite 5.4.0 por defecto
- `README.md` - Actualizado con nueva info

## 🚀 Proceso de Reinstalación Recomendado

### 1. Backup (si hay sistema previo)
```bash
sudo bash /opt/obsea/bin/more/backup_sqlite.sh
```

### 2. Formatear y reinstalar sistema base
```bash
# Reinstalar Raspberry Pi OS
# Configurar red, hostname, etc.
```

### 3. Clonar e instalar
```bash
git clone https://github.com/cartero7/obsea.git
cd obsea
sudo bash bin/00_install_all.sh
```

### 4. Post-instalación
```bash
sudo bash bin/99_post_install.sh
```

### 5. Iniciar servicios
```bash
sudo /opt/obsea/bin/50_run_all.sh
```

### 6. Verificar acceso
```bash
# obsea3-03:
# Web: http://192.168.1.151:8080
# API: http://192.168.1.151:8101
# Usuario: test / test123
```

## ✅ Configuraciones Críticas por Hostname

### obsea3-01
- IP: 192.168.1.152
- API Port: 8101
- Web Port: 8080
- Mode: prod

### obsea3-02
- IP: 192.168.1.147
- API Port: 8101
- Web Port: 8080
- Mode: prod

### obsea3-03
- IP: 192.168.1.151
- API Port: 8101
- Web Port: 8080
- Mode: prod

### obsea3-test
- IP: localhost
- API Port: 9001
- Web Port: 9000
- Mode: dummy

## 📋 Checklist Validación Post-Instalación

- [ ] Git configurado con identidad
- [ ] Servicios corriendo (ps aux | grep -E "uvicorn|vite|hw_supervisor")
- [ ] Web accesible desde navegador
- [ ] Login funciona con test/test123
- [ ] API responde en /docs
- [ ] Logs creándose en /opt/obsea/log/current/
- [ ] Usuario test existe en BD
- [ ] Permisos correctos en /opt/obsea/data
- [ ] Backup inicial creado

## 🔧 Comandos Útiles

### Verificar servicios
```bash
ps aux | grep -E "uvicorn|vite|hw_supervisor"
```

### Ver logs en tiempo real
```bash
tail -f /opt/obsea/log/current/*.log
```

### Reiniciar servicios
```bash
sudo pkill -f "uvicorn|vite|hw_supervisor"
sudo /opt/obsea/bin/50_run_all.sh
```

### Gestión de usuarios
```bash
sudo /opt/obsea/bin/more/obsea_user_admin.py list-users
sudo /opt/obsea/bin/more/obsea_user_admin.py create-user <nombre>
```

### Backup manual
```bash
sudo bash /opt/obsea/bin/more/backup_sqlite.sh
```

## 📚 Documentación Relevante

- `INSTALL_QUICKSTART.md` - Guía rápida
- `docs/USUARIOS_Y_DATOS.md` - Gestión usuarios y backups
- `docs/SETUP.md` - Setup detallado
- `DEPLOYMENT_GUIDE.md` - Despliegue completo
- `bin/README.md` - Scripts disponibles

## ⚠️ Problemas Conocidos y Soluciones

### Node.js 18 vs Vite 7
**Síntoma:** Error "crypto.hash is not a function"  
**Solución:** Ya aplicada en package.json (vite 5.4.0)

### Permisos en /opt/obsea
**Síntoma:** Cannot write to node_modules  
**Solución:** `sudo chown -R $USER:obsea_admins /opt/obsea/webtest`

### Base de datos vacía
**Síntoma:** Usuario test no existe después de reinstalar  
**Solución:** `sudo bash bin/99_post_install.sh`

---

**Estado:** ✅ Todos los cambios pusheados a GitHub  
**Branch:** main  
**Último commit:** 4da9f115  
**Fecha:** 2025-12-10
