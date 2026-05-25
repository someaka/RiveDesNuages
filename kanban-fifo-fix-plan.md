# Kanban FIFO Race Fix — First Draft (Critically Revised)

> **Status**: Draft v1 — incorporates critical findings from code review
> **Author**: Ed + code verification
> **Date**: 2026-05-25

---

## 1. Investigation Summary

### 1.1 Confirmed Failure Modes

**FM1 — ENXIO Silent Swallow** (`hermes_cli/kanban_db.py:2084-2099`)
- `os.open(fifo, O_WRONLY | O_NONBLOCK)` at line 2091 raises `OSError(ENXIO=6)` when no reader connected
- Outer `except Exception: pass` at line 2098-2099 silently drops the notification
- Comment at 2087-2090 documents this as intentional for CI/headless deadlock avoidance, but silent data loss is the side-effect
- `import errno` is **missing** — must be added for typed exception handling

**FM2 — Reader Doesn't Survive Gateway Restart** (`tui_gateway/server.py:196-283`)
- `_start_kanban_fifo_reader()` starts a `daemon=True` thread at line 282
- Called only at session create (line 735) and session init (line 2290)
- `_reset_session_agent()` at line 2159 does NOT restart the reader — only calls `_restart_slash_worker()`
- Gateway process exit kills all daemon threads → reader dies → no consumer for FIFO

**FM3 — Multi-Session Fragility** (architectural)
- Each session starts its own reader thread; they compete for the single FIFO
- The reader that wins dispatches to ALL sessions via DB query (line 233-234) — functionally correct but fragile:
  - If winning reader's DB is locked/crashed, ALL sessions lose the notification
  - Multiple readers opening/closing FIFO repeatedly creates thundering-herd on `open()`

**No fallback exists**: `_start_notification_poller` handles only `process_registry.completion_queue`, not kanban.

### 1.2 Upstream Issues

| Issue | Relevance | Status |
|---|---|---|
| [#31901](https://github.com/NousResearch/hermes-agent/issues/31901) | Kanban notifications lost for failed sends + decomposed child tasks | Open (2026-05-25). **Different root cause** (SendResult soft-failure + subscription inheritance), same symptom class. Complementary fix. |
| [#22995](https://github.com/NousResearch/hermes-agent/issues/22995) | Feature request: push notification on kanban block/crash | Open. Related but about adding notifications, not fixing delivery. |

**Compatibility note**: #31901's fix may touch `list_notify_subs` or `unseen_events_for_sub` — our drain function uses these. Coordinate with PR author (dysources) to avoid merge conflicts.

### 1.3 Existing Code Map

| Symbol | Location | Keep/Remove |
|---|---|---|
| `_KANBAN_FIFO_PATH` | `server.py:138` | **KEEP** — reuse, don't redefine |
| `_cleanup_kanban_fifo()` | `server.py:152` | **KEEP** — atexit handler removes FIFO on shutdown |
| `_format_kanban_notification()` | `server.py:163` | **KEEP** — module-level, callable from new code |
| `_start_kanban_fifo_reader()` | `server.py:196-283` | **REMOVE** — replaced by global reader + queue |
| `_start_kanban_fifo_reader()` call | `server.py:735` | **REPLACE** with `_drain_kanban_queue_for_session()` |
| `_start_kanban_fifo_reader()` call | `server.py:2290` | **REPLACE** with `_drain_kanban_queue_for_session()` |
| Writer block | `kanban_db.py:2084-2099` | **MODIFY** — typed exceptions + logging |

---

## 2. Implementation Plan

### Strategy: Eager Global Reader + Thread-Safe Queue + Writer Logging

The FIFO's zero-polling architecture is sound. Bugs are in lifecycle management (FM2) and error handling (FM1). We repair, not replace.

### Phase 1 — Writer Fix (Low Risk, Ships Independently)

**File**: `hermes_cli/kanban_db.py`

**Prerequisites**: Add `import errno` (currently missing; `import logging` already at line 83).

**Change**: Replace `except Exception: pass` with typed exception handling + structured logging.

```python
# Lines 2083-2099 → replace with:

    if kind in _NOTIFY_KINDS:
        _fifo_path = os.path.expanduser("~/.hermes/tui_kanban.fifo")
        if os.path.exists(_fifo_path):
            try:
                _fd = os.open(_fifo_path, os.O_WRONLY | os.O_NONBLOCK)
            except OSError as e:
                if e.errno == errno.ENXIO:
                    logger.debug("kanban FIFO write skipped: no reader (ENXIO)")
                else:
                    logger.warning("kanban FIFO open failed: %s", e)
                return
            except Exception:
                logger.warning("kanban FIFO open failed (unexpected)", exc_info=True)
                return
            try:
                _fifo = os.fdopen(_fd, "w", encoding="utf-8")
                try:
                    _fifo.write(json.dumps({"task_id": task_id, "kind": kind}) + "\n")
                finally:
                    _fifo.close()
            except Exception:
                try:
                    os.close(_fd)
                except Exception:
                    pass
                logger.warning(
                    "kanban FIFO write failed for task %s (kind=%s)",
                    task_id, kind, exc_info=True
                )
```

**Key design decisions**:
- ENXIO → `logger.debug()` (not warning) — expected in CI/headless, don't alarm operators
- Other OSErrors → `logger.warning()` — unexpected, worth investigating
- Early `return` after open failure — don't attempt fdopen on invalid fd
- Inner `finally` for `_fifo.close()` — ensures fd is released even if write fails

### Phase 2 — Global Reader + Queue (Core Fix)

**File**: `tui_gateway/server.py`

#### 2a: Add global reader infrastructure (~after line 160, near `_cleanup_kanban_fifo`)

```python
# ── Kanban FIFO global reader (one per gateway process) ──────────────────
# Replaces per-session daemon threads.  A single reader drains the FIFO
# and deposits raw events into a thread-safe queue.  Sessions drain their
# relevant events on init and via a periodic 5-second timer.

import queue as _queue_mod

_kanban_event_queue: _queue_mod.Queue = _queue_mod.Queue()
_kanban_global_reader_started: bool = False
_kanban_reader_stop: threading.Event = threading.Event()


def _start_global_kanban_reader() -> None:
    """Start ONE global FIFO reader for the gateway process.

    Idempotent — subsequent calls are no-ops.
    The reader deposits raw {task_id, kind} dicts into _kanban_event_queue.
    """
    global _kanban_global_reader_started
    if _kanban_global_reader_started:
        return
    _kanban_global_reader_started = True

    import json as _json

    def _reader_loop() -> None:
        while not _kanban_reader_stop.is_set():
            try:
                with open(_KANBAN_FIFO_PATH, "r", encoding="utf-8") as _fifo:
                    for _line in _fifo:
                        if _kanban_reader_stop.is_set():
                            return
                        _line = _line.strip()
                        if not _line:
                            continue
                        try:
                            _data = _json.loads(_line)
                        except Exception:
                            continue
                        if _data.get("task_id"):
                            _kanban_event_queue.put(_data)
            except (OSError, IOError):
                _kanban_reader_stop.wait(1.0)  # FIFO missing → retry in 1s

    _t = threading.Thread(
        target=_reader_loop, daemon=True, name="kanban-fifo-global"
    )
    _t.start()
    logger.info("kanban global FIFO reader started")


def _stop_global_kanban_reader() -> None:
    """Signal the global reader to stop (called during gateway shutdown)."""
    _kanban_reader_stop.set()
```

#### 2b: Shared dispatch helper (avoids duplicating 60 lines of DB logic)

Extract from old `_fifo_reader` into a module-level function:

```python
def _dispatch_kanban_event(sid: str, session: dict, task_id: str,
                           cursors: dict) -> None:
    """Query DB for unseen kanban events for task_id and dispatch to session.

    Called from both session-init drain and periodic drain.
    Updates cursors dict in-place to track per-subscription last-seen IDs.
    """
    _FMT_KINDS = {"completed", "blocked", "gave_up", "crashed", "timed_out"}

    try:
        from hermes_cli import kanban_db as _kb

        _conn = _kb.connect()
        try:
            _subs = _kb.list_notify_subs(_conn)
            for _sub in _subs:
                if _sub.get("platform") != "cli":
                    continue
                if _sub["task_id"] != task_id:
                    continue
                _ckey = (task_id, _sub.get("chat_id", sid))
                _last = cursors.get(_ckey, _sub.get("last_event_id", 0))
                _, _events = _kb.unseen_events_for_sub(
                    _conn,
                    task_id=task_id,
                    platform="cli",
                    chat_id=_sub["chat_id"],
                    thread_id=_sub.get("thread_id") or "",
                    kinds=_FMT_KINDS,
                )
                if _events:
                    _max_id = max(
                        _last,
                        max(
                            e.id if hasattr(e, "id") else e["id"]
                            for e in _events
                        ),
                    )
                    cursors[_ckey] = _max_id
                    for _ev in _events:
                        _msg = _format_kanban_notification(_ev, _sub)
                        if _msg:
                            _emit(
                                "status.update",
                                sid,
                                {"kind": "process", "text": _msg},
                            )
                            # Thread-safe submit: use history_lock to prevent
                            # race between session-init drain and periodic drain.
                            _rid = f"__kanban__{int(time.time() * 1000)}"
                            with session["history_lock"]:
                                if session.get("running"):
                                    continue
                                session["running"] = True
                            try:
                                _emit("message.start", sid)
                                _run_prompt_submit(_rid, sid, session, _msg)
                            except Exception:
                                with session["history_lock"]:
                                    session["running"] = False
        finally:
            _conn.close()
    except Exception:
        pass  # Non-fatal — don't break the drain loop
```

**Critical fix applied**: `session["history_lock"]` now wraps the entire `running` check-and-set block (was: bare check outside lock). This prevents the race where session-init drain and periodic drain both see `running=False` and submit simultaneously.

#### 2c: Session drain function (replaces `_start_kanban_fifo_reader` calls)

```python
def _drain_kanban_queue_for_session(sid: str, session: dict) -> None:
    """Drain queued kanban events relevant to this session.

    Non-blocking — drains up to 50 events per call.
    Cursors are stored in session state to survive across drain cycles.
    """
    # CRITICAL: cursors MUST persist across calls to prevent re-delivery.
    # The original _fifo_reader kept cursors in its closure; we store
    # them in session state keyed by (task_id, chat_id).
    if "_kanban_cursors" not in session:
        session["_kanban_cursors"] = {}

    drained = 0
    while drained < 50:
        try:
            _data = _kanban_event_queue.get_nowait()
        except _queue_mod.Empty:
            break
        drained += 1

        _task_id = _data.get("task_id", "")
        if _task_id:
            _dispatch_kanban_event(
                sid, session, _task_id, session["_kanban_cursors"]
            )
```

#### 2d: Periodic drain timer

```python
def _start_kanban_queue_drain_timer() -> None:
    """Periodically drain the global kanban queue for all active sessions.

    Runs every 5 seconds.  No-op when no sessions exist.
    """

    def _drain_loop() -> None:
        while not _kanban_reader_stop.is_set():
            _kanban_reader_stop.wait(5.0)
            if _kanban_reader_stop.is_set():
                return
            if not _sessions:  # Skip when idle
                continue
            for sid, session in list(_sessions.items()):
                try:
                    _drain_kanban_queue_for_session(sid, session)
                except Exception:
                    pass

    _t = threading.Thread(
        target=_drain_loop, daemon=True, name="kanban-queue-drain"
    )
    _t.start()
```

#### 2e: Gateway startup — start global reader + drain timer

In the gateway startup path (near existing `_cleanup_kanban_fifo` registration or in `main()`):

```python
# Start the global kanban FIFO reader (idempotent)
_start_global_kanban_reader()
_start_kanban_queue_drain_timer()
```

#### 2f: Gateway shutdown — stop global reader

Integrate with existing shutdown handler:

```python
# In the gateway shutdown/stop handler:
_stop_global_kanban_reader()
```

#### 2g: Replace per-session reader calls

At line 735 and 2290, replace:
```python
_start_kanban_fifo_reader(sid, _sessions[sid])
```
with:
```python
_drain_kanban_queue_for_session(sid, _sessions[sid])
```

### Phase 3 — Remove Old Per-Session Reader

**File**: `tui_gateway/server.py`

Remove `_start_kanban_fifo_reader()` function (lines 196-283).

**Pre-removal audit** (confirmed):
- Only callers: lines 735 and 2290 (both replaced in Phase 2g)
- `_KANBAN_FIFO_PATH` (line 138): **KEEP** — reused by global reader
- `_cleanup_kanban_fifo()` (line 152): **KEEP** — atexit handler still needed
- `_format_kanban_notification()` (line 163): **KEEP** — used by `_dispatch_kanban_event`

### Phase 4 — Tests

**New file**: `tests/tui_gateway/test_kanban_fifo.py`

| Test | What It Verifies |
|---|---|
| `test_writer_enxio_not_fatal` | ENXIO from writer doesn't crash event write |
| `test_writer_enxio_logged` | ENXIO produces debug log (not warning) |
| `test_global_reader_idempotent` | Double-call starts only one thread |
| `test_queue_drain_empty` | Empty queue → no crash, no side effects |
| `test_queue_drain_with_events` | Queued events dispatched to session |
| `test_cursors_persist_across_drains` | **NEW** — cursors survive across drain cycles (no notification spam) |
| `test_drain_race_safety` | **NEW** — concurrent session-init + periodic drain don't double-submit |
| `test_reader_survives_session_reset` | After `_reset_session_agent()`, global reader still running |
| `test_multiple_sessions_same_event` | Two sessions both receive notification |
| `test_shutdown_stops_reader` | **NEW** — `_stop_global_kanban_reader()` terminates reader thread |

---

## 3. Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| Global reader blocks shutdown | Low | Daemon thread dies with process; `_kanban_reader_stop` set in shutdown handler |
| Queue memory growth | Low | 50 events/tick limit; drain every 5s; `Queue` is bounded by consumption rate |
| Notification re-delivery (cursors bug) | **Fixed** | Cursors stored in `session["_kanban_cursors"]`, persist across drain cycles |
| Race on `session["running"]` | **Fixed** | `history_lock` wraps entire check-and-set block in `_dispatch_kanban_event` |
| Duplicated dispatch logic | **Fixed** | Extracted into `_dispatch_kanban_event()` helper — single source of truth |
| Upstream #31901 merge conflict | Low | Drain function uses same DB API; coordinate with PR author |
| `idle_sessions` drain waste | **Fixed** | `if not _sessions: continue` guard in periodic timer |

---

## 4. Verification Checklist

- [ ] `hermes kanban create` → task completes → TUI shows notification ✅
- [ ] `hermes gateway restart` → new task completes → notification still arrives ✅
- [ ] Two concurrent TUI sessions → both receive kanban notifications ✅
- [ ] CI/headless context → no ENXIO warnings (debug level only) ✅
- [ ] `hermes gateway stop` → reader thread terminates cleanly ✅
- [ ] No notification spam (cursors persist across drain cycles) ✅
- [ ] No regression in existing kanban + gateway tests ✅
- [ ] Thread count stable after gateway startup (no leaks) ✅

---

## 5. Files Changed Summary

| File | Change | Risk |
|---|---|---|
| `hermes_cli/kanban_db.py` | +`import errno`; lines 2084-2099: typed exceptions + logging | Low |
| `tui_gateway/server.py` | +`_start_global_kanban_reader()`, +`_stop_global_kanban_reader()`, +`_dispatch_kanban_event()`, +`_drain_kanban_queue_for_session()`, +`_start_kanban_queue_drain_timer()`; modify lines 735, 2290; remove `_start_kanban_fifo_reader()` (lines 196-283); add startup/shutdown calls | Medium |
| `tests/tui_gateway/test_kanban_fifo.py` | New: 10 test cases | Low |
