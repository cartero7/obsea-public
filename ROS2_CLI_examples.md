# OBSEA3 ROS2 CLI: Ready-to-Run Examples

Copy/paste commands with minimal edits. Only tweak variables marked `# TODO`. Assumes ROS2 Jazzy and the OBSEA workspace are installed at `/opt/obsea/ros2_ws`.

## 1) Operator flow (JWT + lease) — recommended
Standard path with role and lease enforcement.

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Fill in your creds ---
USER="test"       # TODO
PASS="test123"    # TODO
PORT=1            # 1-8
MODE="OFF"        # OFF | 12V | 48V
ROS_DOMAIN_ID=1   # active ROS2 domain

# --- Derive API endpoint ---
API_PORT=$(awk -F= '/^export UVICORN_PORT=/{gsub(/"/,"",$2);print $2;exit}' /opt/obsea/bin/.venv)
API_BASE="http://localhost:${API_PORT:-8101}"

# --- Login and lease ---
TOKEN=$(curl -s -X POST "$API_BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=$USER&password=$PASS" | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

LEASE=$(curl -s -X POST "$API_BASE/ports/$PORT/lease/acquire" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json;print(json.load(sys.stdin).get('lease_id',''))")

echo "TOKEN=${TOKEN:0:12}...  LEASE=$LEASE"

# --- ROS2 call ---
sudo bash -lc "
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=$ROS_DOMAIN_ID
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    '{port: $PORT, mode: \"$MODE\", token: \"$TOKEN\", username: \"$USER\", lease_id: \"$LEASE\"}'
"

# --- Release lease ---
curl -s -X POST "$API_BASE/ports/$PORT/lease/release?lease_id=$LEASE" \
  -H "Authorization: Bearer $TOKEN" >/dev/null
```

## 2) Maintenance flow (system token)
Use offline or when the API is unreachable. Requires `OBSEA3_TOKEN` in `/opt/obsea/bin/secrets.env`.

```bash
#!/usr/bin/env bash
set -euo pipefail

PORT=1
MODE="OFF"
ROS_DOMAIN_ID=1

sudo env PORT="$PORT" MODE="$MODE" ROS_DOMAIN_ID="$ROS_DOMAIN_ID" bash -lc '
  source /opt/obsea/bin/.venv 2>/dev/null   # may define ROS_DOMAIN_ID
  source /opt/obsea/bin/secrets.env             # loads OBSEA3_TOKEN
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-1}
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    "{port: ${PORT}, mode: \"${MODE}\", token: \"${OBSEA3_TOKEN}\"}"
'
```

## 3) One-shot using the token loaded by the running process
Use this when you see `Invalid system token` and want to reuse the token already loaded by the live services.

```bash
#!/usr/bin/env bash
set -euo pipefail

PORT=1
MODE="OFF"
ROS_DOMAIN_ID=1

TOKEN_ACTIVO=$(sudo cat /proc/$(pgrep -f obsea3_api_gateway | head -1)/environ | tr "\0" "\n" | sed -n "s/^OBSEA3_TOKEN=//p")

# The ros2 service call can run without sudo if your user has ROS middleware access on the host.
env PORT="$PORT" MODE="$MODE" ROS_DOMAIN_ID="$ROS_DOMAIN_ID" OBSEA3_TOKEN="$TOKEN_ACTIVO" bash -lc '
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-1}
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    "{port: ${PORT}, mode: \"${MODE}\", token: \"${OBSEA3_TOKEN}\", username: \"\", lease_id: \"\"}"
'

# If your user lacks ROS/middleware permissions, add sudo only to this call:
# sudo env PORT=... bash -lc '... ros2 service call ...'
```

## 4) Quick observability
Activate the environment, then run the checks. Paste as-is in the same session (no sudo):

```bash
set -euo pipefail

# ROS2 + OBSEA workspace
source /opt/ros/jazzy/setup.bash
source /opt/obsea/ros2_ws/install/setup.bash

# Optional runtime vars (ROS_DOMAIN_ID, ports, etc.)
if [ -f /opt/obsea/bin/.venv ]; then
  set -a; source /opt/obsea/bin/.venv; set +a
fi

echo "ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-<undefined>}"

ros2 node list
ros2 topic list | grep /obsea3
ros2 topic echo /obsea3/ports/telemetry
```

- If you get `WARNING: topic ... does not appear to be published yet`, check:
  - `ros2 topic list -t | grep /obsea3/ports/telemetry` to confirm the topic and type.
  - `ros2 node list | grep obsea3` and `ros2 node info <node>`; if no nodes, ensure ROS services are up (systemd) and `ROS_DOMAIN_ID` matches.
  - Wait a few seconds after bringing nodes up and retry the `echo`.

### 4.1) Per-port telemetry quick refs
```bash
# Port 1 metrics (replace p1 → pN)
ros2 topic echo /obsea3/ports/p1/voltage
ros2 topic echo /obsea3/ports/p1/current
ros2 topic echo /obsea3/ports/p1/power
ros2 topic echo /obsea3/ports/p1/vreturn

# Aggregated (all ports)
ros2 topic echo /obsea3/ports/telemetry

# Publish rate check (~5 Hz expected)
ros2 topic hz /obsea3/ports/p1/voltage
```

## 5) Quick notes
- Always run on the control network.
- Match `ROS_DOMAIN_ID` to the active nodes.
- Operator flow requires `username` and `lease_id`; maintenance does not.
- If you change `/opt/obsea/bin/secrets.env`, restart services so they load the new token.
- With `set -euo pipefail`, running `ros2 ...` before sourcing the environment will exit with `ros2: command not found` and drop the session. Source first (see section 4).
