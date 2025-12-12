# Fix: Alarm Deduplication with Recovery Delay

## Problem
The system was sending duplicate alarms every 5 seconds (polling cycle) for the same condition:

```
2025-12-09 10:04:16 AM  [PORT] p1 (v12) overcurrent 1.30A > 1.00A (>0.200s)
2025-12-09 10:04:11 AM  [PORT] p1 (v12) overcurrent 1.10A > 1.00A (>0.200s)
2025-12-09 10:04:06 AM  [PORT] p1 (v12) overcurrent 1.40A > 1.00A (>0.200s)
2025-12-09 10:04:01 AM  [PORT] p1 (v12) overcurrent 1.26A > 1.00A (>0.200s)
...
```

## Root Cause
The alarm deduplication logic was resetting the `*_reported` flag immediately when the condition momentarily cleared, causing repeated alarms when the condition fluctuated.

**Old buggy logic:**
```python
else:
    if not soft_violation:
        st["soft_start"] = None
        st["soft_reported"] = False  # ❌ Immediate reset
```

This caused spam when current fluctuated around the threshold (e.g., 0.98A → 1.02A → 0.99A → 1.05A).

## Solution
Implemented a **recovery delay** mechanism that requires the condition to remain stable (OK) for 10 seconds before resetting the alarm state.

### Changes Made

1. **Added recovery timer fields** to port and bus limit states:
   - `soft_clear_start`, `hard_clear_start`
   - `uv_clear_start`, `ov_clear_start`

2. **Recovery delay logic**:
   ```python
   recovery_delay = 10.0  # seconds
   
   if violation:
       st["clear_start"] = None  # Reset recovery timer
       if not st["reported"]:
           # Report alarm ONCE
           st["reported"] = True
   else:
       if st["reported"]:
           # Start recovery timer if not started
           if st["clear_start"] is None:
               st["clear_start"] = now
           
           # Reset flag only after stable recovery period
           elapsed_clear = now - st["clear_start"]
           if elapsed_clear >= recovery_delay:
               msg = f"alarm CLEARED (stable for {recovery_delay:.1f}s)"
               st["reported"] = False
               st["clear_start"] = None
   ```

3. **Applied to all alarm types**:
   - Port alarms: soft_overcurrent, hard_overcurrent, undervolt, overvolt
   - Bus alarms: soft_overcurrent, hard_overcurrent, undervolt, overvolt

### Files Modified
- `/home/carla/obsea/ros2_ws/src/obsea3_hw_interfaces/obsea3_hw_interfaces/hw_supervisor_node.py`
  - `_reset_port_limit_state()`: Added `*_clear_start` fields
  - `_check_port_limits()`: Implemented recovery delay logic
  - `_check_bus_limits()`: Implemented recovery delay logic

### Behavior After Fix

**Before:**
- Alarm reported every 5s while condition persists
- 100+ duplicate alarms per hour for persistent issues

**After:**
- Alarm reported ONCE when condition first triggers (after trip_delay)
- Recovery message logged after 10s of stable OK condition
- No spam during fluctuations

**Example timeline:**
```
T+0.0s:  Current = 1.30A (above 1.00A threshold)
T+0.2s:  soft_start timer started
T+0.4s:  ALARM REPORTED: "p1 overcurrent 1.30A > 1.00A"
T+5.0s:  Current = 1.25A (still above) → No new alarm (already reported)
T+10.0s: Current = 1.32A (still above) → No new alarm (already reported)
T+15.0s: Current = 0.95A (now OK) → clear_start timer started
T+20.0s: Current = 0.92A (stable OK for 5s)
T+25.0s: Current = 0.90A (stable OK for 10s) → "overcurrent CLEARED" logged
T+30.0s: Current = 1.10A (above again) → NEW ALARM reported
```

## Testing

### Build
```bash
cd /home/carla/obsea
bash bin/30_build_ros2_ws.sh
```

### Run
```bash
bash bin/50_run_all.sh
```

### Monitor Logs
```bash
# Watch for alarm patterns
tail -f /opt/obsea/log/current/hw_supervisor.log | grep -E "(overcurrent|CLEARED)"

# Check active alarms
bash bin/more/check_alarms.sh
```

### Expected Results
- No repeated alarms for same persistent condition
- Single alarm when condition triggers
- Recovery message after 10s stable period
- New alarm if condition re-triggers after recovery

## Configuration
The recovery delay is hardcoded at 10 seconds. To adjust:

```python
# In hw_supervisor_node.py, line ~988
recovery_delay = 10.0  # Change this value (seconds)
```

Consider making this configurable via `limits.yaml` if needed:
```yaml
global:
  alarm_recovery_delay_s: 10.0
```

## Impact
- ✅ Eliminates alarm spam in web interface
- ✅ Reduces Zabbix alert noise
- ✅ SQLite database growth reduced
- ✅ Clearer alarm lifecycle tracking
- ✅ Maintains detection sensitivity (trip_delay still enforced)
