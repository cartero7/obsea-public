# OBSEA3 Basic Usage

This guide assumes the system is already installed and running.
Examples are provided for curl and the ROS2 CLI.

## Prerequisites

- API reachable at `API_BASE`
- A user account with the right role and port permissions
- Python 3 for JSON parsing (or replace with `jq` if preferred)

## Health checks

```bash
API_BASE="http://localhost:8100"  # adjust to your configured port

curl -s "$API_BASE/health"
curl -s "$API_BASE/ready"
```

If `ready` returns 503, the API is up but telemetry is not ready yet.

## REST API (curl)

### Login and token

```bash
API_BASE="http://localhost:8100"  # adjust to your configured port

TOKEN=$(curl -s -X POST "$API_BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=test&password=test123" | \
  python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")
```

### Check your permissions

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/me/ports" | python3 -m json.tool
```

### Telemetry and status

```bash
curl -s "$API_BASE/telemetry" | python3 -m json.tool
curl -s "$API_BASE/ports/1/telemetry" | python3 -m json.tool
curl -s "$API_BASE/gpio_state" | python3 -m json.tool
curl -s "$API_BASE/telemetry/alarms" | python3 -m json.tool
```

### Python helper (REST)

Standalone Python example (no repo access required). Save as `api_example.py`:

```python
#!/usr/bin/env python3
import requests, json, sys

API_BASE = "http://localhost:8100"  # adjust to your API
USER = "test"
PASS = "test123"

def login(api_base, username, password):
    resp = requests.post(
        f"{api_base}/auth/login",
        data={"username": username, "password": password},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        timeout=10,
    )
    resp.raise_for_status()
    token = resp.json().get("access_token")
    if not token:
        raise RuntimeError("No access_token in response")
    return token

def get_telemetry(api_base, token):
    resp = requests.get(
        f"{api_base}/telemetry",
        headers={"Authorization": f"Bearer {token}"},
        timeout=10,
    )
    resp.raise_for_status()
    return resp.json()

def set_mode(api_base, token, port, mode, lease_id=None):
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    if lease_id:
        headers["X-Lease-Id"] = lease_id
    resp = requests.post(
        f"{api_base}/ports/{port}/mode",
        headers=headers,
        json={"mode": mode},
        timeout=10,
    )
    resp.raise_for_status()
    return resp.json()

def main():
    try:
        token = login(API_BASE, USER, PASS)
        print(f"[OK] Logged in, token length={len(token)}")
        tel = get_telemetry(API_BASE, token)
        print("[OK] Telemetry snippet:")
        print(json.dumps(tel, indent=2)[:400], "...")
        res = set_mode(API_BASE, token, port=1, mode="48V")
        print("[OK] set_mode response:", res)
    except requests.HTTPError as e:
        print(f"HTTP error: {e.response.status_code} {e.response.text}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Run with: `python3 api_example.py` (requires `pip install requests`).

### Change port mode

Valid modes are `OFF`, `12V`, `48V`.

```bash
curl -s -X POST "$API_BASE/ports/1/mode" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"12V"}' | python3 -m json.tool
```

### Lease workflow (exclusive control)

Acquire a lease:

```bash
LEASE=$(curl -s -X POST "$API_BASE/ports/1/lease/acquire" \
  -H "Authorization: Bearer $TOKEN" | \
  python3 -c "import sys, json; print(json.load(sys.stdin).get('lease_id',''))")
```

Use the lease:

```bash
curl -s -X POST "$API_BASE/ports/1/mode" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Lease-Id: $LEASE" \
  -H "Content-Type: application/json" \
  -d '{"mode":"OFF"}'
```

Release the lease:

```bash
curl -s -X POST "$API_BASE/ports/1/lease/release?lease_id=$LEASE" \
  -H "Authorization: Bearer $TOKEN"
```

## ROS2 CLI

### Environment

```bash
# Load runtime config if available (sets ROS_DOMAIN_ID and other vars)
# Adjust paths to your installation
if [ -f /opt/obsea/bin/.venv ]; then
  set -a; source /opt/obsea/bin/.venv; set +a
fi

source /opt/ros/jazzy/setup.bash       # or your ROS2 setup path
source /opt/obsea/ros2_ws/install/setup.bash  # adjust to your install
```

### Inspect nodes and topics

```bash
ros2 node list
ros2 topic list | grep /obsea3
ros2 topic echo /obsea3/ports/telemetry
```

### Service call (advanced)

Use the REST API for normal operations. ROS2 service calls are intended for controlled environments.

```bash
export ROS_DOMAIN_ID=90   # ajusta al dominio de tu sistema
ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
  "{port: 1, mode: '12V', token: '<SYSTEM_TOKEN>'}"
```

Notes:
- `<SYSTEM_TOKEN>` is `OBSEA3_TOKEN` from `/opt/obsea/bin/secrets` (root only). This bypasses user/roles.
- If ROS2 auth is configured to use JWT, pass a user JWT in `token` and the `username` + `lease_id` fields accordingly.
  ```bash
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    "{port: 1, mode: '12V', token: '<USER_JWT>', username: 'admin', lease_id: '1-XXXX'}"
  ```
- Always source the runtime before calling:
  ```bash
  source /opt/obsea/bin/secrets
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  export ROS_DOMAIN_ID=90   # ajusta al valor de tu entorno
  ```

One-shot example (system token):
```bash
sudo bash -lc '
  export ROS_DOMAIN_ID=90
  source /opt/obsea/bin/secrets
  source /opt/ros/jazzy/setup.bash
  source /opt/obsea/ros2_ws/install/setup.bash
  ros2 service call /obsea3/ports/set_mode_srv obsea3_interfaces/srv/SetPortMode \
    "{port: 1, mode: \"12V\", token: \"${OBSEA3_TOKEN}\"}"
'
```

- Lease = operator exclusion. The system token respects existing leases but does not create new ones. If you need a lock from ROS2, use JWT + username + lease_id or create a system lease explicitly for long operations.

Full operator-mode example (JWT + lease): see `bin/tools/ros2_operator_example.py` in the repo. It:
- Configure `API_BASE`, `USER`, `PASSWORD`, `PORT`, `MODE`, `ROS_DOMAIN_ID` at the top.
- Logs in via REST, acquires a lease, then calls the ROS2 service with JWT + username + lease_id (sources ROS overlays inside).
- Run: `python3 bin/tools/ros2_operator_example.py` (requires `pip install requests`).

## Troubleshooting

- If you do not see nodes, check `ROS_DOMAIN_ID` and make sure it matches the running system.
- If `ready` returns 503, wait for the hardware supervisor to publish telemetry.
- If a port command fails, verify your role and port permissions.
