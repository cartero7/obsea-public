# OBSEA3 ROS2 CLI Guide

Use the ROS2 CLI only when you are on the control network and understand the hardware impact. REST is preferred for routine operations.

For ready-to-run examples (operator flow, maintenance flow, and token one-shot), see `docs-public/ROS2_CLI_examples.md`. This page stays focused on context and troubleshooting.

## Prerequisites
- API already installed and running (see `UVICORN_PORT` in `/opt/obsea/bin/.venv`).
- ROS2 Jazzy and the OBSEA workspace sourced.
- A valid token (system token for maintenance, or user JWT for operator actions).

### Prepare the environment
```bash
# Optional: load runtime variables if present (defines ROS_DOMAIN_ID, etc.)
if [ -f /opt/obsea/bin/.venv ]; then
  set -a; source /opt/obsea/bin/.venv; set +a
fi

source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash
```

## Which token to use?
- **Operator flow (recommended):** login via REST to get a user JWT, acquire a lease, and pass `token`, `username`, and `lease_id` to the service call. This enforces roles and leases.
- **Maintenance flow (last resort):** root-only `OBSEA3_TOKEN` from `/opt/obsea/bin/secrets`. It respects existing leases but bypasses user roles. Do not expose it.

## Activate a port via ROS2 (operator flow, tested)
1) Set base variables and obtain a JWT (uses the API only to log in and get a lease):
```bash
API_PORT=$(awk -F= '/^export UVICORN_PORT=/{gsub(/"/,"",$2);print $2;exit}' /opt/obsea/bin/.venv)
API_BASE="http://localhost:${API_PORT:-8101}"
USER="test"      # replace
PASS="test123"   # replace
PORT=1           # valid range: 1-8
MODE="OFF"       # modes: OFF | 12V | 48V (use OFF by default)

TOKEN=$(curl -s -X POST "$API_BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=$USER&password=$PASS" | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

2) Acquire a lease for the port (recommended):
```bash
LEASE=$(curl -s -X POST "$API_BASE/ports/$PORT/lease/acquire" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json;print(json.load(sys.stdin).get('lease_id',''))")
```

3) Call the ROS2 service with JWT + username + lease (example ROS_DOMAIN_ID=1):
```bash
sudo bash -lc "
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=1
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    '{port: $PORT, mode: \"$MODE\", token: \"$TOKEN\", username: \"$USER\", lease_id: \"$LEASE\"}'
"
```

4) Release the lease when done:
```bash
curl -s -X POST "$API_BASE/ports/$PORT/lease/release?lease_id=$LEASE" \
  -H "Authorization: Bearer $TOKEN"
```

## Maintenance flow (system token)
Use when the API is unreachable or you need an offline ROS-only call. The token must match the one loaded by the running services. Valid ports: 1–8. Modes: OFF | 12V | 48V.
```bash
sudo bash -lc '
  source /opt/obsea/bin/.venv 2>/dev/null   # may set ROS_DOMAIN_ID
  source /opt/obsea/bin/secrets             # provides OBSEA3_TOKEN
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-1}  # set to the domain used by the running nodes
  echo "ROS_DOMAIN_ID=$ROS_DOMAIN_ID"
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    "{port: 1, mode: \"OFF\", token: \"${OBSEA3_TOKEN}\"}"
'
```

### If it fails (token or domain issues)
- Confirm the service is visible: `ros2 service list | grep set_mode`.
- Check the token actually loaded by the running process (in case it differs from `/opt/obsea/bin/secrets`):
  ```bash
  sudo cat /proc/$(pgrep -f obsea3_api_gateway | head -1)/environ | tr '\0' '\n' | grep '^OBSEA3_TOKEN='
  ```
  Use that value in the call if it differs.
- Ensure you are on the same ROS domain as the running nodes: `echo $ROS_DOMAIN_ID` vs process launch arguments; set `export ROS_DOMAIN_ID=<value>` accordingly.
- After changing `/opt/obsea/bin/secrets`, restart the services so they load the new token (e.g., systemd units for API/supervisor).
- If sudo access is unavailable or the system token still fails, use the operator flow (login + lease) which is already verified to work.

#### One-shot using the token loaded by the running process
Use this when `Invalid system token` appears and you want to reuse the live token without restarting services:
```bash
TOKEN_ACTIVO=$(sudo cat /proc/$(pgrep -f obsea3_api_gateway | head -1)/environ | tr '\0' '\n' | sed -n 's/^OBSEA3_TOKEN=//p')

env ROS_DOMAIN_ID="${ROS_DOMAIN_ID:-1}" OBSEA3_TOKEN="${TOKEN_ACTIVO}" bash -lc "
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    '{port: 1, mode: \"OFF\", token: \"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"${OBSEA3_TOKEN}'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"'\"', username: \"\", lease_id: \"\"}'
"

# Si tu usuario no tiene permisos ROS/middleware, añade sudo solo a esta llamada:
# sudo env ROS_DOMAIN_ID=... OBSEA3_TOKEN=... bash -lc '... ros2 service call ...'
```
If this works, you may want to update `/opt/obsea/bin/secrets` with `TOKEN_ACTIVO` and restart services so future calls can use the standard maintenance command.

## Observability
```bash
ros2 node list
ros2 topic list | grep /obsea3
ros2 topic echo /obsea3/ports/telemetry
```

## Troubleshooting
- No response / wrong domain: ensure `ROS_DOMAIN_ID` matches the running system.
- 403 or auth errors: verify you are using the right token type; for operator flow you must pass `token`, `username`, and `lease_id`.
- Lease errors: another operator holds the lease; release or wait.
- Connection refused: confirm the API and ROS2 processes are running on the target host.
- `Invalid system token`: the running service loaded a different `OBSEA3_TOKEN`. Either restart services after updating `/opt/obsea/bin/secrets`, or read the token from the process environment (e.g., `/proc/$(pgrep -f obsea3_api_gateway)/environ`) and call with that value.
