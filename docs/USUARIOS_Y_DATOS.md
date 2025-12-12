# Gestión de Usuarios y Protección de Datos

## ⚠️ IMPORTANTE: Protección de Datos

Antes de ejecutar migraciones o reinstalaciones que puedan afectar las bases de datos:

### 1. Hacer Backup Manual

```bash
# Backup de las bases de datos
sudo bash /opt/obsea/bin/more/backup_sqlite.sh

# Verificar backups
ls -lh /opt/obsea/db_archive/$(date +%Y/%m)/
```

### 2. Crear Usuario de Prueba

Después de reinstalar o migrar:

```bash
# Opción 1: Modo interactivo (recomendado)
sudo bash /opt/obsea/bin/03_create_users.sh

# Opción 2: Línea de comandos directa
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py create-user test
# (introduce password: test123)

# Listar usuarios
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py list-users
```

### 3. Gestión de Usuarios

```bash
# Modo interactivo (recomendado) - wrapper bash
sudo bash /opt/obsea/bin/03_create_users.sh

# Modo interactivo - Python directo
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py

# Comandos CLI (ejemplos)
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py list-users
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py create-user <nombre>
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py set-password <nombre>
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py assign-role <usuario> <rol>
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py grant-port <usuario> <puerto>
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py list-user-ports <usuario>
```

**Roles disponibles:** `admin`, `operator`, `viewer`, `superadmin`

**Ejemplo completo de creación de usuario admin:**
```bash
# Crear usuario
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py create-user operador

# Asignar rol admin
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py assign-role operador admin

# Asignar todos los puertos (1-8)
for port in {1..8}; do 
  sudo python3 /opt/obsea/bin/more/obsea_user_admin.py grant-port operador $port
done

# Verificar
sudo python3 /opt/obsea/bin/more/obsea_user_admin.py list-user-ports operador
```

## Bases de Datos

El sistema usa dos bases de datos SQLite:

- `/opt/obsea/data/auth.db` - Usuarios y autenticación
- `/opt/obsea/data/obsea3_api.db` - Datos de la API (puertos, eventos, etc.)

## Backups Automáticos

Los backups se guardan en: `/opt/obsea/db_archive/YYYY/MM/`

Para configurar backups automáticos con cron:

```bash
# Editar crontab
sudo crontab -e

# Agregar línea para backup diario a las 3 AM
0 3 * * * /opt/obsea/bin/more/backup_sqlite.sh
```

## Restaurar desde Backup

```bash
# Listar backups disponibles
ls -lh /opt/obsea/db_archive/$(date +%Y/%m)/

# Restaurar (ejemplo)
sudo cp /opt/obsea/db_archive/2025/12/auth.20251210T100000Z.db /opt/obsea/data/auth.db
sudo cp /opt/obsea/db_archive/2025/12/obsea3_api.20251210T100000Z.db /opt/obsea/data/obsea3_api.db

# Reiniciar servicios
sudo pkill -f "uvicorn|vite|hw_supervisor"
sudo /opt/obsea/bin/50_run_all.sh
```

## Checklist Pre-Migración

Antes de ejecutar scripts que modifiquen el sistema:

- [ ] Hacer backup manual con `backup_sqlite.sh`
- [ ] Verificar que los backups existan
- [ ] Anotar usuarios existentes (`list-users`)
- [ ] Si es posible, exportar configuraciones importantes

## Usuario de Prueba por Defecto

**Usuario:** test  
**Password:** test123  
**Rol:** viewer (solo lectura)

⚠️ Cambiar o eliminar en producción.
