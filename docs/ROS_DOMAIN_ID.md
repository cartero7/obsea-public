# ROS_DOMAIN_ID - Guía Crítica

## ⚠️ PROBLEMA MÁS COMÚN EN OBSEA3

**Síntoma**: "Service call timed out", "Node not discovered", nodos no se ven entre sí.

**Causa**: Diferentes terminales usando diferentes `ROS_DOMAIN_ID`.

**Solución**: **TODOS** los terminales deben tener el **MISMO** `ROS_DOMAIN_ID`.

---

## ¿Qué es ROS_DOMAIN_ID?

Es una variable de entorno que ROS2 usa para aislar redes de nodos. Piensa en ello como un "canal de radio":
- Nodos en dominio 0 solo ven otros nodos en dominio 0
- Nodos en dominio 42 solo ven otros nodos en dominio 42
- Valores válidos: **0-232**

**REGLA DE ORO**: Si tus nodos/servicios no se descubren, 99% es un problema de ROS_DOMAIN_ID.

---

## Configuración en OBSEA3

### Valor Default
El sistema usa `ROS_DOMAIN_ID=0` (configurado en `bin/.venv`)

### Verificar ROS_DOMAIN_ID Actual

```bash
# En cualquier terminal
echo $ROS_DOMAIN_ID
# Si está vacío o es diferente a 0, hay problema
```

### Configuración Automática (desde v2025.12.09)

El script `bin/00_interactive_setup.sh` **añade automáticamente** `ROS_DOMAIN_ID` a tu `~/.bashrc`.

**Para instalaciones nuevas**: No necesitas hacer nada, se configura solo.

**Para instalaciones existentes**: Ejecuta una vez:
```bash
echo "export ROS_DOMAIN_ID=0  # OBSEA3 default domain" >> ~/.bashrc
source ~/.bashrc
```

### Verificar Configuración

**Opción 1: Cargar desde .venv (RECOMENDADO)**
```bash
set -a
source /opt/obsea/bin/.venv
set +a

echo $ROS_DOMAIN_ID  # Debe mostrar: 0
```

**Opción 2: Configurar manualmente**
```bash
export ROS_DOMAIN_ID=0
```

**Opción 3: Añadir a ~/.bashrc (permanente)**
```bash
echo "export ROS_DOMAIN_ID=0" >> ~/.bashrc
source ~/.bashrc
```

---

## Flujo de Trabajo Correcto

### Terminal 1: Lanzar hw_supervisor
```bash
cd /opt/obsea
set -a; source bin/.venv; set +a  # Carga ROS_DOMAIN_ID=0
bash bin/50_run_all.sh
```

### Terminal 2: Probar comandos ROS2
```bash
# ⚠️ PRIMERO: Configurar mismo domain
export ROS_DOMAIN_ID=0

# LUEGO: Source workspace
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash

# AHORA SÍ: Comandos ROS2
ros2 node list
ros2 service list
ros2 topic echo /obsea3/port/1/telemetry
```

### Terminal 3: Usar API
```bash
# La API no necesita ROS_DOMAIN_ID porque se conecta internamente
# Pero si vas a usar comandos ros2 en el mismo terminal:
export ROS_DOMAIN_ID=0
```

---

## Checklist de Diagnóstico

Si los nodos no se ven:

### 1. Verificar ROS_DOMAIN_ID en CADA terminal
```bash
# Terminal donde está hw_supervisor
ps aux | grep hw_supervisor  # Ver PID
cat /proc/<PID>/environ | tr '\0' '\n' | grep ROS_DOMAIN_ID
# Ejemplo: ROS_DOMAIN_ID=0

# Terminal donde haces comandos
echo $ROS_DOMAIN_ID
# DEBE SER EL MISMO VALOR
```

### 2. Verificar que los nodos están corriendo
```bash
export ROS_DOMAIN_ID=0  # Asegurar mismo domain
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
ros2 node list
# Esperado: /hw_supervisor, /api_gateway, /zabbix_bridge_node
```

### 3. Verificar servicios disponibles
```bash
ros2 service list | grep obsea3
# Esperado: /obsea3/ports/set_mode_srv
```

### 4. Si aún no funciona: Reiniciar con domain explícito
```bash
# Detener todo
pkill -f obsea3
pkill -f hw_supervisor

# Verificar limpieza
ps aux | grep obsea3

# Relanzar con domain explícito
export ROS_DOMAIN_ID=0
cd /opt/obsea
bash bin/50_run_all.sh
```

---

## Casos de Uso Especiales

### Múltiples Sistemas OBSEA3 en Misma Red

Si tienes 2+ sistemas OBSEA3 en la misma red física:

**Sistema 1**:
```bash
# bin/.venv
export ROS_DOMAIN_ID=10
```

**Sistema 2**:
```bash
# bin/.venv
export ROS_DOMAIN_ID=20
```

Esto evita que los nodos de un sistema interfieran con el otro.

### Testing Simultáneo

Para probar 2 configuraciones en la misma máquina:

**Instancia producción**:
```bash
export ROS_DOMAIN_ID=0
# Lanzar servicios normales
```

**Instancia test**:
```bash
export ROS_DOMAIN_ID=100
# Lanzar servicios de prueba
```

---

## Errores Comunes

### Error: "Service call timed out"
```bash
# Verificar domain
echo $ROS_DOMAIN_ID

# Si está vacío o diferente:
export ROS_DOMAIN_ID=0

# Reintentar
ros2 service call /obsea3/ports/set_mode_srv ...
```

### Error: "Node '/hw_supervisor' not found"
```bash
# Terminal 1: Verificar domain del nodo corriendo
ps aux | grep hw_supervisor  # Ver PID
cat /proc/<PID>/environ | tr '\0' '\n' | grep ROS_DOMAIN_ID

# Terminal 2: Configurar MISMO domain
export ROS_DOMAIN_ID=<valor_del_paso_anterior>
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
ros2 node list  # Ahora debe aparecer
```

### Error: "No executable found"
Este NO es problema de ROS_DOMAIN_ID, es problema de sourcing:
```bash
# Solución
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
```

---

## Referencias Rápidas

### Template para Nuevos Terminales
```bash
#!/bin/bash
# Copiar esto al inicio de cada terminal ROS2

export ROS_DOMAIN_ID=0
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash

# O en una línea:
# export ROS_DOMAIN_ID=0 && source /opt/ros/jazzy/setup.bash && source /opt/obsea/ros2_ws/install/setup.bash
```

### Alias Útil para ~/.bashrc
```bash
alias obsea-env='export ROS_DOMAIN_ID=0 && source /opt/ros/jazzy/setup.bash && source /opt/obsea/ros2_ws/install/setup.bash && echo "✓ OBSEA env loaded (ROS_DOMAIN_ID=$ROS_DOMAIN_ID)"'

# Uso:
# obsea-env
# ros2 node list
```

---

## Resumen Ejecutivo

1. **SIEMPRE** configurar `ROS_DOMAIN_ID` **ANTES** de usar comandos `ros2`
2. **TODOS** los terminales deben usar el **MISMO** valor (default: 0)
3. Verificar con `echo $ROS_DOMAIN_ID` antes de cada sesión
4. Si los nodos no se ven → 99% es problema de domain diferente
5. Usar `set -a; source bin/.venv; set +a` para configurar automáticamente

---

**Última actualización**: 2025-12-09  
**Prioridad**: 🔴 CRÍTICA - Leer antes de trabajar con ROS2
