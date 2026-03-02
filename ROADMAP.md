# Roadmap — pycozmo2

> Current state and work plan to transform the library into a stable, debuggable, and maintainable tool.

---

## 1. Current State

### 1.1 General Architecture

```
run.py          → entry point (context manager connect())
client.py       → Client class: robot state + high-level API  (588 lines, God Object)
conn.py         → Connection + SendThread + ReceiveThread (UDP, sliding window)
event.py        → event dispatching system (threading)
anim_controller.py → animation / audio queue
logger.py       → 6 named loggers, no centralized configuration
exception.py    → minimal exception hierarchy
protocol_*      → Cozmo protocol encoding/decoding
```

### 1.2 Issues Identified by Severity

#### 🔴 CRITICAL — Connection Stability

| # | File | Line(s) | Issue |
|---|------|---------|-------|
| C1 | `conn.py` `SendThread.run()` | ~84 | ~~`except Exception: pass` — **all errors are silently swallowed**; if sending fails, the thread continues as if nothing happened, ping stops being sent, Cozmo restarts after 5 s.~~ **✅ Fixed — log + `stop_flag` + `_on_send_thread_error` callback to `Connection`.** |
| C2 | `conn.py` `Connection.run()` | ~388 | ~~The ping is sent in the main loop that also processes events. If an event handler takes time (e.g. `_initialize_robot` doing `time.sleep(0.5)`), the ping can be delayed by several seconds.~~ **✅ Fixed — dedicated `PingThread`.** |
| C3 | `client.py` `_initialize_robot()` | ~142 | `time.sleep(0.5)` inside the event dispatching thread = blocks the entire `Connection.run()` loop, ping included. |
| C4 | `conn.py` | — | **No reconnection logic**: after `_on_disconnect`, the state goes back to `IDLE` and nothing happens. The user application crashes or hangs. |
| C5 | `conn.py` `Connection` | ~330 | `PING_INTERVAL = 0.5 s` — correct in theory, but no watchdog verifies that the ping is actually being sent. |

#### 🟠 IMPORTANT — Logging

| # | File | Issue |
|---|------|-------|
| L1 | `conn.py` `_on_packet_received` | Every received packet is logged at `DEBUG` via `logger_protocol`. The robot sends `RobotState` ~10 Hz: **~600 lines/min** in DEBUG mode, drowning out real issues. |
| L2 | `client.py` `_on_robot_state` | Dispatches `EvtRobotStateUpdated` at 10 Hz + DEBUG logs of status flags on every transition: unreadable in production. |
| L3 | `run.py` `setup_basic_logging` | All loggers share the **same handler** and the same configuration — impossible to fine-tune levels without modifying the code. |
| L4 | `conn.py` `log_stats` | Stats are only logged every 60 s (`STATS_INTERVAL`). No way to query them on-demand. |
| L5 | General | No `%(funcName)s` or `%(lineno)d` in the log format — impossible to know which function emitted a message without grep. |

#### 🟡 QUALITY — Code and Maintainability

| # | File | Issue |
|---|------|-------|
| Q1 | `client.py` | God Object, 588 lines: robot state + command API + camera handling + animation. Needs to be split. |
| Q2 | `setup.py` | Uses `distutils.version.LooseVersion` — **deprecated in Python 3.10, removed in 3.12**. The library can only be installed on Python ≤ 3.11. |
| Q3 | `setup.py` | `python_requires=">=3.6.0"` — Python 3.6 has been EOL since 2021. |
| Q4 | General | Partial type annotations (a few `Optional`, nothing on return types). Incompatible with `mypy --strict`. |
| Q5 | `conn.py` | `SendThread` and `ReceiveThread` do not expose their error state (`is_alive()` is not enough). |
| Q6 | `event.py` `Dispatcher` | No timeout mechanism on blocking handlers. |
| Q7 | General | No `__slots__` on protocol dataclasses → non-negligible memory overhead at 10 Hz. |

#### 🔵 TESTS

| # | Issue |
|---|-------|
| T1 | No tests for the `conn.py` layer (connection, ping, reconnection, timeout). |
| T2 | No tests for `client.py` (minimal network mock). |
| T3 | `requirements-dev.txt` does not include `pytest-cov`, `pytest-timeout`, `pytest-mock`. |

---

## 2. Work Plan by Phase

### Phase 1 — Connection Stability (top priority)

**Goal: Cozmo no longer restarts unexpectedly.**

#### 1.a Isolate the ping in its own thread

```
conn.py → class PingThread(Thread)
```

- Dedicated thread that sends a `Ping` every `PING_INTERVAL` seconds **independently** of the event loop.
- Built-in watchdog: if no ACK has been received for `> 3 s`, emit `EvtConnectionWarning`.
- If no ACK for `> 4.5 s` (close to the 5 s limit), attempt a reconnection.

#### 1.b Remove `time.sleep()` from event handlers

```
client.py → _initialize_robot() → use a Timer or a deferred Task
```

- Replace `time.sleep(0.5)` with an async timer or a `threading.Timer`.

#### 1.c Fix `except Exception: pass` in `SendThread`

```python
# BEFORE
except Exception:
    pass

# AFTER
except Exception as e:
    logger.error("SendThread crashed: %s", e, exc_info=True)
    # Trigger the connection loss event
    self._on_send_error(e)
```

#### 1.d Add reconnection logic

```
conn.py → Connection
```

- New state: `RECONNECTING`
- When `_on_disconnect` is called unexpectedly (not following `disconnect()`), emit `EvtConnectionLost` and start an automatic reconnection attempt (exponential backoff: 1 s, 2 s, 4 s, max 30 s).
- New parameter `auto_reconnect: bool = True` on `Connection`.

#### 1.e Add `EvtConnectionLost` and `EvtConnectionRestored`

```
event.py → new events
```

---

### Phase 2 — Usable Logging

**Goal: useful logs without noise.**

#### 2.a Filter `RobotState` from the default log

- `RobotState` is a high-frequency telemetry packet.
- Remove it from the `DEBUG` logs of `_on_packet_received` **by default**.
- Add a `TRACE` level (or use `logging.DEBUG - 5`) for raw packets.

```python
# conn.py
TRACE = 5
logging.addLevelName(TRACE, "TRACE")

# Usage
logger_protocol.log(TRACE, "Got  %s", pkt)  # for RobotState, Ping, etc.
logger_protocol.debug("Got  %s", pkt)       # for significant packets only
```

#### 2.b Improve the log format

```python
# run.py
formatter = logging.Formatter(
    fmt="%(asctime)s.%(msecs)03d [%(name)-18s] %(levelname)-8s %(funcName)s:%(lineno)d — %(message)s",
    datefmt="%H:%M:%S")
```

#### 2.c Per-logger environment variables

Add to `setup_basic_logging` the reading of:
- `PYCOZMO_LOG_LEVEL` (existing)
- `PYCOZMO_PROTOCOL_LOG_LEVEL` (existing)
- `PYCOZMO_ROBOT_LOG_LEVEL` (existing)
- `PYCOZMO_CONN_LOG_LEVEL` (new — specific to conn.py)
- `PYCOZMO_ANIM_LOG_LEVEL` (new)

#### 2.d Make `log_stats()` available on-demand

```python
# conn.py
# callable from outside + called automatically on EvtConnectionLost
```

#### 2.e Log connection transitions as first-class events

```
CONNECTING → CONNECTED   : logger.info("Connected to Cozmo (FW %s)", fw_version)
CONNECTED  → IDLE        : logger.warning("Connection lost after %.1f s", uptime)
RECONNECTING             : logger.info("Reconnecting (attempt %d)...", attempt)
```

---

### Phase 3 — Code Cleanup

#### 3.a Fix Python 3.10+ compatibility

```python
# setup.py — replace distutils with packaging
from packaging.version import Version

python_requires=">=3.14"
```

#### 3.b Split `client.py`

Proposed split:

| New file | Responsibility |
|----------|----------------|
| `client.py` | Orchestration, lifecycle, `connect()`/`disconnect()` |
| `robot_state.py` | Robot state data (pose, speed, battery…) |
| `command_api.py` | High-level commands (`drive_wheels`, `set_head_angle`…) |
| `camera.py` (existing) | Extended with image chunk decoding |

#### 3.c Complete type annotations

- Goal: `mypy --strict` with no errors on the `conn`, `client`, `event`, `exception` modules.
- Add `py.typed` for PEP 561 compatibility.

#### 3.d Replace ad-hoc `dict`s with dataclasses

```python
# BEFORE (client.py)
self.connected_objects[pkt.object_id] = {
    "factory_id": pkt.factory_id,
    "object_type": pkt.object_type,
}

# AFTER
@dataclass
class ConnectedObject:
    factory_id: int
    object_type: ObjectType
```

---

### Phase 4 — Tests

#### 4.a Network test infrastructure

- Create `tests/conftest.py` with a `mock_cozmo_server` fixture (local UDP socket that simulates the robot).
- Test the following scenarios:
  - Normal connection
  - Connection timeout
  - Unexpected disconnection → automatic reconnection
  - Delayed ping (lag simulation)
  - Sliding window overflow

#### 4.b Minimum coverage target

| Module | Target |
|--------|--------|
| `conn.py` | 80 % |
| `client.py` | 60 % |
| `window.py` | 95 % (already partially tested) |
| `frame.py` | 90 % |

#### 4.c Add to `requirements-dev.txt`

```
pytest-cov
pytest-timeout
pytest-mock
packaging
```

---

### Phase 5 — Documentation

- Google-style docstrings on all public methods.
- `ARCHITECTURE.md` updated with the thread diagram.
- `DEBUGGING.md`: guide on how to enable verbose logging, read stats, and diagnose a disconnection.
- Enriched examples: `examples/minimal.py` with explicit handling of connection events.

---

## 3. Suggested Execution Order

```
Phase 1a  →  Phase 1b  →  Phase 1c   (block immediate regressions)
     ↓
Phase 1d  →  Phase 1e               (reconnection)
     ↓
Phase 2a  →  Phase 2b  →  Phase 2c  (readable logs)
     ↓
Phase 3a  →  Phase 3b               (modernization)
     ↓
Phase 4a  →  Phase 4b               (safety net)
     ↓
Phase 3c  →  Phase 3d  →  Phase 5   (polish)
```

---

## 4. What MUST NOT Be Changed (for now)

- The UDP protocol itself (`frame.py`, `protocol_encoder.py`, `window.py`) — it works and any change risks breaking compatibility with the robot.
- The event system (`event.py` / `Dispatcher`) — to be extended, not replaced.
- The `CozmoAnim/` and `audiokinetic/` files — stable, low risk.

---

## 5. Success Metrics

- [ ] Session duration > 30 min without robot restart.
- [ ] `mypy --strict pycozmo/conn.py pycozmo/client.py` → 0 errors.
- [ ] `pytest --cov=pycozmo --cov-fail-under=70` → passes.
- [ ] DEBUG mode produces no more than 10 lines/s in steady state (excluding `TRACE`).
- [ ] Automatic reconnection working after a 10 s WiFi outage.
