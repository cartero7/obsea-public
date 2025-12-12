# Sistema de Alarmas OBSEA3

## Descripción General

El sistema de alarmas monitoriza límites de seguridad en tiempo real y registra eventos en la base de datos SQLite. Cada alarma tiene un ciclo de vida completo con detección, registro y recuperación.

## Tipos de Alarmas

### Por Puerto (1-8)
- **soft_overcurrent**: Sobrecorriente suave (>límite, <tiempo para hard)
- **hard_overcurrent**: Sobrecorriente dura (>límite durante >tiempo configurado)
- **undervolt**: Subtensión (voltaje por debajo del mínimo)
- **overvolt**: Sobretensión (voltaje por encima del máximo)

### Por Bus de Alimentación
- **v12_overcurrent**: Sobrecorriente en bus 12V
- **v48_overcurrent**: Sobrecorriente en bus 48V
- **v5_overcurrent**: Sobrecorriente en bus 5V (auxiliar)
- **v12_undervolt/overvolt**: Límites de tensión en bus 12V
- **v48_undervolt/overvolt**: Límites de tensión en bus 48V

## Configuración de Límites

Archivo: `/opt/obsea/data/limits.yaml`

```yaml
ports:
  1:
    max_current: 1.0      # Amperios máximos
    min_voltage: 11.0     # Voltios mínimos
    max_voltage: 13.0     # Voltios máximos
    overcurrent_duration: 0.2  # Segundos antes de hard alarm

buses:
  v12:
    max_current: 5.0
    min_voltage: 11.5
    max_voltage: 12.5
```

**Hot-reload**: El hw_supervisor recarga automáticamente este archivo cuando detecta cambios (monitoriza mtime cada 5 segundos).

## Consultar Alarmas Activas

### Método 1: Script CLI Simple
```bash
cd /opt/obsea
bash bin/more/check_alarms.sh
```

Salida:
```
118 alarma(s) detectada(s) en las últimas 24 horas

Puerto: 1 | Tipo: soft_overcurrent | Nivel: WARN | Duración: 5s
Mensaje: [PORT] p1 (v12) overcurrent 1.32A > 1.00A (>0.200s)
Timestamp: 2025-12-09 08:17:02

Puerto: 1 | Tipo: hard_overcurrent | Nivel: ERROR | Duración: 5s
Mensaje: [PORT] p1 (v12) overcurrent 1.32A > 1.00A (>0.200s)
Timestamp: 2025-12-09 08:17:02
...
```

### Método 2: Formato JSON
```bash
bash bin/more/check_alarms.sh --json
```

Salida JSON para procesamiento programático:
```json
[
  {
    "id": 12345,
    "timestamp": "2025-12-09T08:17:02",
    "level": "WARN",
    "source": "hw_supervisor",
    "message": "[PORT] p1 (v12) overcurrent 1.32A > 1.00A (>0.200s)",
    "port": 1,
    "alarm_type": "soft_overcurrent",
    "duration_seconds": 5
  }
]
```

### Método 3: Monitoreo Continuo
```bash
bash bin/more/check_alarms.sh --watch
```

Actualiza cada 5 segundos (útil para monitoreo en tiempo real).

### Método 4: API REST Directa
```bash
# 1. Obtener token JWT
TOKEN=$(curl -s -X POST http://localhost:8101/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' \
  | jq -r .access_token)

# 2. Consultar alarmas
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8101/telemetry/alarms | jq
```

**Nota**: El endpoint NO usa prefijo `/api/v1/` (diseño de esta API).

## Base de Datos

### Localización
```
/opt/obsea/data/obsea3_api.db
```

### Tabla: system_events
```sql
CREATE TABLE system_events (
    id INTEGER PRIMARY KEY,
    timestamp TEXT,
    level TEXT,           -- INFO, WARN, ERROR
    source TEXT,          -- hw_supervisor, safety_manager, etc.
    message TEXT,
    port INTEGER,
    extra TEXT            -- JSON adicional
);
```

### Consulta SQL Directa
```bash
sqlite3 /opt/obsea/data/obsea3_api.db << EOF
SELECT 
    timestamp, level, port, message 
FROM system_events 
WHERE level IN ('WARN', 'ERROR')
    AND timestamp > datetime('now', '-24 hours')
ORDER BY timestamp DESC 
LIMIT 20;
EOF
```

## Lógica de Deduplicación

### Estado por Puerto
```python
_port_limit_state = {
    1: {
        "soft_reported": False,
        "hard_reported": False,
        "undervolt_reported": False,
        "overvolt_reported": False
    },
    # ... puertos 2-8
}
```

### Estado por Bus
```python
_bus_limit_state = {
    "v12": {
        "soft_reported": False,
        "hard_reported": False,
        "undervolt_reported": False,
        "overvolt_reported": False
    },
    # v48, v5
}
```

### Flujo de Eventos

1. **Primera detección**: Log WARN/ERROR + flag `*_reported = True`
2. **Detecciones subsiguientes**: Silenciadas (no log)
3. **Recuperación**: Log INFO con "CLEARED" + flag `*_reported = False`
4. **Nueva violación**: Vuelve a logear (ciclo reinicia)

### Ejemplo de Logs
```
2025-12-09 08:17:02 | WARN  | hw_supervisor | [PORT] p1 (v12) overcurrent 1.32A > 1.00A
2025-12-09 08:17:07 | INFO  | hw_supervisor | [PORT] p1 (v12) overcurrent CLEARED
2025-12-09 08:18:15 | WARN  | hw_supervisor | [PORT] p1 (v12) overcurrent 1.32A > 1.00A
```

## Integración con Frontend (Futuro)

### Endpoint WebSocket
Para streaming en tiempo real:
```javascript
const ws = new WebSocket('ws://localhost:8101/api/v1/telemetry/stream');
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    // Filtrar alarmas: data.level === 'WARN' || data.level === 'ERROR'
};
```

### Componente React Sugerido
```javascript
import { useEffect, useState } from 'react';

function AlarmsPanel() {
    const [alarms, setAlarms] = useState([]);
    
    useEffect(() => {
        fetch('/telemetry/alarms', {
            headers: { 'Authorization': `Bearer ${getToken()}` }
        })
        .then(res => res.json())
        .then(setAlarms);
    }, []);
    
    return (
        <div>
            <h2>Alarmas Activas ({alarms.length})</h2>
            {alarms.map(a => (
                <AlarmCard key={a.id} alarm={a} />
            ))}
        </div>
    );
}
```

## Troubleshooting

### Alarma no se limpia
**Síntoma**: Alarma persiste en `/telemetry/alarms` después de resolver condición.

**Causa**: Flag `*_reported` no se reseteó correctamente.

**Solución**:
1. Verificar logs: `tail -f /opt/obsea/log/current/hw_supervisor.log`
2. Buscar mensaje "CLEARED" correspondiente
3. Si no aparece, revisar lógica en `_check_port_limits()` o `_check_bus_limits()`

### Logs duplicados persisten
**Síntoma**: Múltiples logs idénticos en segundos sucesivos.

**Causa**: Diccionario de estado no inicializado o lógica de deduplicación fallando.

**Verificación**:
```bash
# Contar logs duplicados recientes
sqlite3 /opt/obsea/data/obsea3_api.db << EOF
SELECT message, COUNT(*) as count 
FROM system_events 
WHERE timestamp > datetime('now', '-1 hour')
GROUP BY message 
HAVING count > 5
ORDER BY count DESC;
EOF
```

**Solución**: Reiniciar hw_supervisor para reinicializar estados:
```bash
# Detener
pkill -f hw_supervisor_node

# Iniciar (desde 50_run_all.sh o manualmente)
source ros2_ws/install/setup.bash
ros2 run obsea3_hw_interfaces hw_supervisor_node
```

### API devuelve 401 Unauthorized
**Causa**: Token JWT inválido o expirado.

**Solución**:
```bash
# Regenerar token
TOKEN=$(curl -s -X POST http://localhost:8101/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' \
  | jq -r .access_token)

# Verificar token
echo $TOKEN
```

## Referencias

- **Código fuente alarmas**: `ros2_ws/src/obsea3_hw_interfaces/obsea3_hw_interfaces/hw_supervisor_node.py`
- **API endpoint**: `ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/main.py:815`
- **Función consulta**: `ros2_ws/src/obsea3_api_gateway/obsea3_api_gateway/actions_log.py:get_active_alarms()`
- **Script CLI**: `bin/more/check_alarms.sh`
- **Configuración límites**: `data/limits.yaml`
