# CONTRACT.md — wrangle-imprint JSON Interface Specification

This document defines the exact JSON shapes exchanged between the three layers of wrangle-imprint:

```
OpenClaw / dispatcher
        ↕  [OpenClaw ↔ IMPRINT contract]
    imprint.py
        ↕  [IMPRINT ↔ wrangle contract]
    wrangle.py
        ↕  ADB → Android device
```

---

## 1. OpenClaw ↔ IMPRINT

### Request (CLI / dispatcher calling IMPRINT)

IMPRINT is invoked as a subprocess or CLI. The dispatcher passes an intent string and optional flags:

```
python imprint.py ask "<intent>" [--dry] [--confirmed] [--queue]
```

| Flag | Effect |
|------|--------|
| `--dry` | Resolve and print steps without executing |
| `--confirmed` | Skip destructive-action safety prompt |
| `--queue` | Enqueue for deferred execution instead of running now |

For programmatic use, IMPRINT can also be called via the queue API:

```json
{ "intent": "open youtube and search for lo-fi hip hop", "dry_run": false }
```

---

### Response shapes

IMPRINT always writes a single JSON object to stdout. `source` tells you which path was taken.

#### Cache hit — trusted plan

```json
{
  "source": "cache",
  "plan_id": "a1b2c3d4e5f6a7b8",
  "similarity": 0.91,
  "tokens": 0,
  "duration_ms": 3200,
  "steps_taken": 5
}
```

#### Cache hit — pending plan (being confirmed)

```json
{
  "source": "cache",
  "plan_id": "a1b2c3d4e5f6a7b8",
  "similarity": 0.81,
  "tokens": 0,
  "duration_ms": 4100,
  "steps_taken": 6
}
```

#### LLM path — new plan, succeeded and stored

```json
{
  "source": "llm",
  "plan_id": "b2c3d4e5f6a7b8c9",
  "tokens": 812,
  "duration_ms": 7400,
  "steps_taken": 5,
  "trusted": false
}
```

`trusted: true` means the plan hit the confirmation threshold immediately (only possible when `IMPRINT_CONFIRM=1`).

#### Mid-task replan (step failure → LLM recovery)

```json
{
  "source": "replan",
  "plan_id": "a1b2c3d4e5f6a7b8",
  "similarity": 0.0,
  "tokens": 540,
  "duration_ms": 9100,
  "steps_taken": 8
}
```

#### Failure response

Any source can fail. Failed responses always include `"success": false` and an `"error"` field:

```json
{
  "source": "cache",
  "plan_id": "a1b2c3d4e5f6a7b8",
  "success": false,
  "error": "step 3 (tap): target_not_found"
}
```

```json
{
  "source": "llm",
  "success": false,
  "error": "llm_key_missing",
  "tokens": 0
}
```

#### Dry run response

```json
{
  "source": "cache",
  "plan_id": "a1b2c3d4e5f6a7b8",
  "similarity": 0.88,
  "dry_run": true
}
```

#### Queued response

```json
{
  "queued": true,
  "intent": "turn off wifi"
}
```

---

### Error codes

| Code | Meaning |
|------|---------|
| `adb_unavailable` | ADB connection could not be established |
| `state_parse_error` | UIAutomator XML dump failed to parse |
| `target_not_found` | Element resolver found no match for step target |
| `launch_failed` | App could not be launched |
| `input_failed` | ADB input command failed |
| `screen_drift` | Screen state didn't match expected hash before execution |
| `unsafe_action_blocked` | Destructive action blocked; use `--confirmed` to override |
| `max_steps_exceeded` | Task exceeded `IMPRINT_MAX_STEPS` |
| `task_timeout` | Task exceeded `IMPRINT_TIMEOUT` seconds |
| `llm_key_missing` | `CEREBRAS_KEY` not set and LLM path was needed |

---

### Plans JSON (programmatic inspection)

```
python imprint.py plans-json
```

Returns an array of plan objects:

```json
[
  {
    "id": "a1b2c3d4e5f6a7b8",
    "template": "open {app} and search for {message}",
    "trusted": 1,
    "confirm_count": 3,
    "hits": 12,
    "failures": 1,
    "param_slots": "[\"app\", \"message\"]",
    "last_used": "2026-03-10T14:22:01",
    "created_at": "2026-02-28T09:11:44"
  }
]
```

---

## 2. IMPRINT ↔ wrangle

IMPRINT calls wrangle as a subprocess:

```
python wrangle.py get_state --task "<intent>"
python wrangle.py do_action --json '<action_json>'
```

---

### get_state response

```json
{
  "screen_hash": "a3f9b2e14c7d",
  "state_signature": "8fa1c3b29e01",
  "foreground_app": "com.google.android.youtube",
  "screen_summary": "14 ranked elements from 87 raw UI nodes",
  "raw_count": 87,
  "elements": [
    {
      "id": 12,
      "label": "Search YouTube",
      "text": "Search YouTube",
      "content-desc": "Search YouTube",
      "resource-id": "search_edit_text",
      "class": "EditText",
      "role": "input",
      "clickable": true,
      "editable": true,
      "focusable": true,
      "focused": false,
      "enabled": true,
      "scrollable": false,
      "checkable": false,
      "checked": false,
      "selected": false,
      "x": 540,
      "y": 112,
      "x_norm": 0.5,
      "y_norm": 0.0479,
      "center": [540, 112],
      "bounds": "[0,88][1080,136]",
      "score": 14.5
    }
  ]
}
```

On error:

```json
{
  "screen_hash": "error",
  "screen_summary": "XML parse error: ...",
  "foreground_app": "unknown",
  "elements": [],
  "raw_count": 0
}
```

---

### do_action request

All actions share an `"action"` field. Each action type has its own required fields:

#### tap
```json
{ "action": "tap", "x": 540, "y": 1200, "reason": "tapping search bar" }
```

#### type
```json
{ "action": "type", "text": "lo-fi hip hop", "reason": "typing search query" }
```

#### swipe
```json
{ "action": "swipe", "x1": 540, "y1": 1400, "x2": 540, "y2": 600, "ms": 300, "reason": "scrolling down" }
```

#### scroll (shorthand)
```json
{ "action": "scroll", "direction": "down", "amount": 800, "reason": "revealing more results" }
```

#### keyevent
```json
{ "action": "keyevent", "key": "KEYCODE_ENTER", "reason": "submitting search" }
```

#### back
```json
{ "action": "back", "reason": "returning to previous screen" }
```

#### launch
```json
{ "action": "launch", "app": "youtube", "reason": "opening YouTube", "url": null }
```

`app` can be a known alias (e.g. `"chrome"`, `"youtube"`, `"settings"`), a package name, or a full component string.

#### done
```json
{ "action": "done", "reason": "task complete" }
```

---

### do_action response

#### Success
```json
{ "ok": true, "executed": "tap", "x": 540, "y": 1200 }
```
```json
{ "ok": true, "executed": "type", "text": "lo-fi hip hop" }
```
```json
{ "ok": true, "executed": "launch", "app": "youtube", "url": null }
```

#### Failure
```json
{ "ok": false, "error": "tap requires integer x and y" }
```
```json
{ "ok": false, "error": "missing action" }
```
```json
{ "ok": false, "error": "unsupported action: hover" }
```

---

### Element targeting (IMPRINT → wrangle)

When IMPRINT sends steps from the LLM to `execute_one_step`, steps can include a `target` field for element-based resolution instead of raw coordinates:

```json
{ "action": "tap", "target": { "text": "Search YouTube" }, "reason": "open search", "delay": 1.0 }
{ "action": "tap", "target": { "id": "search_edit_text" }, "reason": "open search", "delay": 1.0 }
{ "action": "tap", "target": { "desc": "Search" }, "reason": "open search", "delay": 1.0 }
{ "action": "tap", "target": "Send", "reason": "send message", "delay": 0.5 }
```

IMPRINT resolves the target against the current `elements` array from `get_state` and substitutes `x`/`y` before calling `do_action`. If resolution fails, the step falls back to any `x`/`y` already in the step, or fails with `target_not_found`.

Resolution priority: `id` (score +12) > `text` (score +10–20) > `desc` (score +8). Clickable elements get a +5 bonus.

---

## 3. Queue table schema

The offline queue stores tasks for deferred execution:

```sql
CREATE TABLE queue (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    ts           TEXT,           -- ISO timestamp of enqueue
    intent       TEXT NOT NULL,  -- raw natural language intent
    dry_run      INTEGER DEFAULT 0,
    status       TEXT DEFAULT 'pending',  -- pending | done | failed | error
    result       TEXT,           -- JSON result blob (truncated to 500 chars)
    error        TEXT,           -- error string on failure
    processed_at TEXT            -- ISO timestamp of execution
);
```

Flush pending queue entries:

```bash
python imprint.py flush
```

---

## 4. Step log schema

Every executed step is recorded:

```sql
CREATE TABLE step_log (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    ts             TEXT,
    plan_id        TEXT,
    step_num       INTEGER,
    action         TEXT,
    target         TEXT,
    success        INTEGER,   -- 1 or 0
    screen_changed INTEGER,   -- 1 if screen hash changed after action
    error          TEXT,
    duration_ms    INTEGER
);
```

---

## 5. Versioning

| Component | Version |
|-----------|---------|
| imprint.py | 2.0.0 |
| wrangle.py | 2.0.0 |
| Contract spec | 2.0.0 |

Breaking changes to this contract must bump the major version of both files and this document.
