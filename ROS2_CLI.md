# ROS2 CLI (Public)

Este documento resume el uso de ROS2 CLI para OBSEA3 en entornos publicos.
Se ha eliminado el ejemplo que extraia `OBSEA3_TOKEN` desde `/proc/.../environ`.
Si necesitas un token de sistema, usa el fichero `/opt/obsea/bin/secrets` (root).

## Entorno
```bash
# Carga variables de runtime (incluye ROS_DOMAIN_ID)
set -a; source /opt/obsea/bin/.venv; set +a

# ROS2 overlays
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
```

## Verificar nodos y topics
```bash
ros2 node list
ros2 topic list | grep /obsea3
ros2 topic echo /obsea3/ports/telemetry
```

## Service call (system token, debug)
```bash
SYSTEM_TOKEN=$(sudo grep -m1 "OBSEA3_TOKEN" /opt/obsea/bin/secrets | cut -d'"' -f2)

ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '$SYSTEM_TOKEN'}"
```

## Service call (JWT + lease)
```bash
ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '<USER_JWT>', username: 'admin', lease_id: '1-XXXX'}"
```

## Notas
- `ROS_DOMAIN_ID` debe coincidir con el sistema en ejecucion.
- Para uso normal, prefiere la API REST.
- El token de sistema respeta leases existentes pero no crea nuevos.
