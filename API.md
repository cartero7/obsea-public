# OBSEA3 REST API — Ready Examples

Public-facing, copy/paste-friendly guide for the OBSEA3 REST API. Prefer the API over ROS2 CLI for routine operations: it enforces JWT auth, roles, and leases.

## Quick setup
```bash
# Resolve API port from install (fallback 8100)
API_PORT=$(awk -F= '/^export UVICORN_PORT=/{gsub(/"/,"",$2);print $2;exit}' /opt/obsea/bin/.venv 2>/dev/null)
API_BASE="http://localhost:${API_PORT:-8100}"

# Credentials (replace)
USER="test"    # TODO
PASS="test123" # TODO
```

## 1) Health checks
```bash
curl -s "$API_BASE/health"
curl -s "$API_BASE/ready"
```
`ready` may be `503` while telemetry boots.

## 2) Login & whoami
```bash
TOKEN=$(curl -s -X POST "$API_BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=$USER&password=$PASS" |
  python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

curl -s -H "Authorization: Bearer $TOKEN" "$API_BASE/me/ports" | python3 -m json.tool
```

## 3) Telemetry & status
```bash
curl -s "$API_BASE/telemetry" | python3 -m json.tool          # all ports
curl -s "$API_BASE/ports/1/telemetry" | python3 -m json.tool  # one port
curl -s "$API_BASE/gpio_state" | python3 -m json.tool
curl -s "$API_BASE/telemetry/alarms" | python3 -m json.tool
```

## 4) Change port mode (with lease recommended)
Valid modes: `OFF`, `12V`, `48V`.
```bash
PORT=1
MODE="OFF"  # use OFF by default to avoid energizing hardware

# Acquire lease (recommended for shared systems)
LEASE=$(curl -s -X POST "$API_BASE/ports/$PORT/lease/acquire" \
  -H "Authorization: Bearer $TOKEN" |
  python3 -c "import sys,json;print(json.load(sys.stdin).get('lease_id',''))")

# Set mode
curl -s -X POST "$API_BASE/ports/$PORT/mode" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Lease-Id: $LEASE" \
  -H "Content-Type: application/json" \
  -d "{\"mode\":\"$MODE\"}" | python3 -m json.tool

# Release lease
curl -s -X POST "$API_BASE/ports/$PORT/lease/release?lease_id=$LEASE" \
  -H "Authorization: Bearer $TOKEN"
```

## 5) Lease heartbeat / status
```bash
curl -s "$API_BASE/ports/$PORT/lease" | python3 -m json.tool
curl -s -X POST "$API_BASE/lock/port_${PORT}/heartbeat" -H "Authorization: Bearer $TOKEN"
```

## 6) Config endpoints
```bash
curl -s "$API_BASE/config/instruments" | python3 -m json.tool
curl -s "$API_BASE/config/limits" | python3 -m json.tool
# Update (admin only)
# curl -X PUT "$API_BASE/config/instruments" -H "Authorization: Bearer $TOKEN" \
#      -H "Content-Type: text/plain" --data-binary @instruments.yaml
```

## 7) Logs & audit
```bash
curl -s "$API_BASE/ports/1/last_action" | python3 -m json.tool
curl -s "$API_BASE/ports/actions" | python3 -m json.tool
curl -s "$API_BASE/logs/system" | python3 -m json.tool
```

## 8) Full bash workflow (telemetry → change → telemetry)
```bash
#!/usr/bin/env bash
set -euo pipefail
API_BASE="${API_BASE:-http://localhost:${API_PORT:-8100}}"
PORT=1

TOKEN=$(curl -s -X POST "$API_BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=$USER&password=$PASS" |
  python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

echo "=== Telemetry before ==="
curl -s "$API_BASE/ports/$PORT/telemetry" | python3 -m json.tool

echo -e "\\n=== Set 12V ==="
curl -s -X POST "$API_BASE/ports/$PORT/mode" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"mode":"12V"}' | python3 -m json.tool

sleep 5

echo -e "\\n=== Telemetry after ==="
curl -s "$API_BASE/ports/$PORT/telemetry" | python3 -m json.tool
```

## 9) Python example (requests)
```python
#!/usr/bin/env python3
import time, requests, os

API_BASE = os.getenv("API_BASE", "http://localhost:8100")
USER = os.getenv("API_USER", "test")
PASS = os.getenv("API_PASS", "test123")
PORT = int(os.getenv("PORT", "1"))

token = requests.post(f"{API_BASE}/auth/login",
                      data={"username": USER, "password": PASS}).json()["access_token"]
hdrs = {"Authorization": f"Bearer {token}"}

def telem():
    return requests.get(f"{API_BASE}/ports/{PORT}/telemetry").json()

print("Before:", telem())
requests.post(f"{API_BASE}/ports/{PORT}/mode", headers=hdrs, json={"mode": "12V"})
for i in range(5):
    time.sleep(5)
    t = telem()
    print(f"[{i+1}] mode={t['mode']} V={t['voltage']} I={t['current']} P={t['power']}")
```

## 10) Python example with lease (requests)
```python
#!/usr/bin/env python3
import requests, os

API_BASE = os.getenv("API_BASE", "http://localhost:8100")
USER = os.getenv("API_USER", "test")
PASS = os.getenv("API_PASS", "test123")
PORT = int(os.getenv("PORT", "1"))

# Login
token = requests.post(f"{API_BASE}/auth/login",
                      data={"username": USER, "password": PASS}).json()["access_token"]
hdrs = {"Authorization": f"Bearer {token}"}

# Acquire lease (recommended)
lease = requests.post(f"{API_BASE}/ports/{PORT}/lease/acquire", headers=hdrs).json().get("lease_id", "")
print(f"Lease acquired: {lease}")

# Set mode with lease
resp = requests.post(f"{API_BASE}/ports/{PORT}/mode",
                     headers={**hdrs, "X-Lease-Id": lease},
                     json={"mode": "12V"}).json()
print("Set mode:", resp)

# Read telemetry
telem = requests.get(f"{API_BASE}/ports/{PORT}/telemetry").json()
print("Telemetry:", telem)

# (Optional) Return to OFF for safety
requests.post(f"{API_BASE}/ports/{PORT}/mode",
              headers={**hdrs, "X-Lease-Id": lease},
              json={"mode": "OFF"})

# Release lease
requests.post(f"{API_BASE}/ports/{PORT}/lease/release", headers=hdrs, params={"lease_id": lease})
print("Lease released")
```

## 11) Interactive docs
- Swagger UI: `$API_BASE/docs`
- ReDoc: `$API_BASE/redoc`
- OpenAPI JSON: `$API_BASE/openapi.json`

## Troubleshooting
- **401/403**: bad credentials or missing port permissions (`/me/ports` to verify).
- **Connection refused**: check `UVICORN_PORT` in `/opt/obsea/bin/.venv` and ensure the API service is running.
- **Lease errors**: someone else holds it; release or wait.
- **ready returns 503**: telemetry still initializing; retry after a few seconds.
- **JWT expired**: log in again (tokens typically last ~15 minutes).
