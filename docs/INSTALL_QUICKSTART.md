# OBSEA3 - Guía de Instalación Rápida

## ⚡ Instalación desde cero

**Requisitos**: Ubuntu 24.04 LTS (Noble Numbat) - x86_64 o ARM64

### 1. Preparación del sistema (como root o con sudo)

```bash
# Clonar repositorio
cd ~
git clone https://github.com/cartero7/obsea.git
cd obsea

# Ejecutar instalación completa
sudo bash bin/00_install_all.sh
```

El script `00_install_all.sh` ejecutará automáticamente:
- Instalación de dependencias del sistema
- Instalación de dependencias Python
- Construcción del workspace ROS2
- Configuración de usuarios y grupos
- Migraciones de base de datos

### 2. Configuración interactiva

```bash
# Configurar parámetros del sistema
bash bin/00_interactive_setup.sh
```

Este script creará el archivo `.venv` con la configuración según el hostname de la máquina.

### 3. Crear usuario de prueba

```bash
# Crear usuario test (password: test123)
sudo /opt/obsea/bin/more/obsea_user_admin.py create-user test
# Cuando pida password: test123
```

### 4. Iniciar servicios

```bash
# Iniciar todos los servicios
sudo /opt/obsea/bin/50_run_all.sh
```

Esto iniciará:
- **HW Supervisor** (ROS2 nodes)
- **API Gateway** (puerto 8101)
- **Web Frontend** (puerto 8080)

## 🌐 Acceso al sistema

Después de la instalación, los servicios estarán disponibles en:

- **Web Frontend**: `http://<IP_DEL_SISTEMA>:8080`
- **API Gateway**: `http://<IP_DEL_SISTEMA>:8101`

**Para obtener la IP de tu sistema:**
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1 | awk '{print $2}' | cut -d/ -f1
```

**Modo dummy/test:**
- Web: `http://localhost:9000`
- API: `http://localhost:9001`

**Usuario por defecto:** `test` / `test123`

## 📝 Notas importantes

### Versión de Node.js
El sistema requiere Node.js 18.x. Si Vite 7.x está instalado, se debe hacer downgrade:

```bash
cd /opt/obsea/webtest
sudo npm install vite@^5.4.0 --save-dev --legacy-peer-deps
```

### Estructura de puertos

Cada máquina tiene asignados diferentes puertos según el hostname. Ver `/opt/obsea/bin/.venv` después de ejecutar `00_interactive_setup.sh`.

### Configuración de red

El frontend tiene hardcodeada la IP del API en `webtest/src/views/App.jsx`:
- Debe coincidir con la IP de la máquina
- Puerto debe ser el `UVICORN_PORT` configurado (normalmente 8101)

### Logs

Los logs se guardan en:
```
/opt/obsea/log/YYYY/MM/DD/
```

Accesos rápidos:
```
/opt/obsea/log/current/api_gateway.log
/opt/obsea/log/current/hw_supervisor.log  
/opt/obsea/log/current/web.log
```

### Backups de bases de datos

**IMPORTANTE:** Antes de reinstalar o migrar:

```bash
# Hacer backup
sudo bash /opt/obsea/bin/more/backup_sqlite.sh

# Verificar
ls -lh /opt/obsea/db_archive/$(date +%Y/%m)/
```

Ver documentación completa en: `docs/USUARIOS_Y_DATOS.md`

## 🔧 Troubleshooting

### Error "vite: not found"
```bash
cd /opt/obsea/webtest
sudo npm install
# Si persiste, downgrade vite:
sudo npm install vite@^5.4.0 --save-dev --legacy-peer-deps
```

### Error de permisos
```bash
sudo chown -R $USER:obsea_admins /opt/obsea/webtest
sudo chown -R $USER:obsea_admins /opt/obsea/data
```

### Servicios no inician
```bash
# Ver logs
tail -f /opt/obsea/log/current/*.log

# Matar procesos zombies
sudo pkill -f "uvicorn|vite|hw_supervisor"

# Reiniciar
sudo /opt/obsea/bin/50_run_all.sh
```

### Base de datos vacía después de reinstalar
```bash
# Recrear usuario test
sudo /opt/obsea/bin/more/obsea_user_admin.py create-user test

# O restaurar desde backup
sudo cp /opt/obsea/db_archive/YYYY/MM/auth.*.db /opt/obsea/data/auth.db
sudo cp /opt/obsea/db_archive/YYYY/MM/obsea3_api.*.db /opt/obsea/data/obsea3_api.db
```

## 📚 Documentación adicional

- `docs/SETUP.md` - Configuración detallada
- `docs/DEPLOYMENT_GUIDE.md` - Guía de despliegue
- `docs/USUARIOS_Y_DATOS.md` - Gestión de usuarios y backups
- `docs/DESARROLLO.md` - Guía para desarrolladores
- `bin/README.md` - Descripción de scripts

## ✅ Checklist de instalación

- [ ] Sistema base instalado (00_install_all.sh)
- [ ] Configuración interactiva completada (00_interactive_setup.sh)
- [ ] Usuario test creado
- [ ] Servicios iniciados correctamente
- [ ] Web accesible desde navegador
- [ ] Login funciona con usuario test
- [ ] Backup automático configurado (opcional)
- [ ] Git configurado con identidad correcta

```bash
git config user.name "Tu Nombre"
git config user.email "tu@email.com"
```

---

**¡Listo para usar!** 🎉

Para cualquier problema, consultar logs en `/opt/obsea/log/current/` o revisar documentación en `docs/`.
