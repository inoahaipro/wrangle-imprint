# wrangle-imprint v2

**wrangle-imprint v2** is a local "phone brain" for Android — a two-part system that lets a language model teach your phone to do tasks once, then replay them forever from local memory with zero API calls.

- **`wrangle.py`** — A text-first Android UI controller that can tap, type, scroll, launch apps, and read the screen via ADB + UIAutomator.
- **`imprint.py`** — A Local Agentic Model (LAM) that turns natural language intents into reusable plans stored in SQLite and executed by `wrangle.py`.

> **How it works:** The LLM teaches a plan **once**. After that, IMPRINT replays it from local memory with **zero tokens** for known tasks.

**Example:**
```bash
python imprint.py ask "open youtube.com in chrome and search for lo-fi hip hop then open the first result" --confirmed
```
First run → LLM plans + executes → plan cached.  
Later runs → IMPRINT replays the plan without touching the LLM.

---

## Architecture

```
OpenClaw (chat / dispatcher)
      │   natural language intent
            ▼
               imprint.py   ←── SQLite (~/.imprint/memory.db)
                     │   JSON action steps
                           ▼
                              wrangle.py   ←── ADB → physical Android device
                                    │   JSON results
                                          ▼
                                             imprint.py   → summary / stats
                                             ```

                                             The exact JSON shapes between IMPRINT and wrangle are documented in [`CONTRACT.md`](./CONTRACT.md).

---

## Status

Early-stage but functional. Currently supports:

- Ranking and summarizing the current Android screen via UIAutomator
- Executing structured actions: `tap` / `type` / `scroll` / `swipe` / `keyevent` / `launch` / `back` / `done`
- Caching successful plans in SQLite and reusing them for similar intents
- Mid-task replan + per-step retries when a step fails on live UI
- Queue + flush mode (`imprint.py ask --queue`, `imprint.py flush`) for temporary ADB outages
- Guarding destructive operations (delete / uninstall / purchase / etc.) unless explicitly confirmed with `--confirmed`

**Tested on:**

| Component | Version |
|-----------|---------|
| Device | Samsung S22+ (Android) |
| Environment | Termux |
| Python | 3.13 (3.11+ should work) |
| ADB | `localhost:34371` |

---

## Prerequisites

### 1. OpenClaw on Android

For running OpenClaw + agents on Android:

- https://github.com/AidanPark/openclaw-android#step-4-install-openclaw

Install via:

```bash
curl -sL myopenclawhub.com/install | bash && source ~/.bashrc
```

This gives you a working OpenClaw gateway, model provider config, and the Termux environment this project expects.

### 2. Termux + Python

```bash
pkg update
pkg install git python openssh
pip install --upgrade pip
pip install requests
```

### 3. ADB over localhost

`wrangle.py` talks to ADB on `localhost:34371` by default:

```python
ADB_PORT = os.environ.get("ADB_PORT", "34371")
ADB      = f"adb -s localhost:{ADB_PORT}"
```

You must have `adbd` listening on that port. How you configure this depends on your device and OpenClaw-Android setup.

### 4. ADBKeyboard IME

Required for safe text injection:

1. Install the [ADBKeyboard APK](https://github.com/senzhk/ADBKeyBoard) on your phone.
2. Register it as an IME:

```bash
adb shell ime set com.android.adbkeyboard/.AdbIME
```

`wrangle.py` temporarily switches to ADBKeyboard for `type` actions and then restores your original keyboard automatically.

### 5. LLM Backend (v2 behavior)

IMPRINT now plans through your **OpenClaw session** (default `OPENCLAW_SESSION=main`) by calling:

```bash
openclaw agent --session-id main --message "..." --json
```

That means IMPRINT uses whatever default model/provider OpenClaw is configured with (for example OpenClaw's default GPT-OSS setup), instead of requiring direct Cerebras configuration in `imprint.py`.

```bash
export OPENCLAW_SESSION=main   # optional override
```

`wrangle.py` still supports direct Cerebras calls for browser/native text reasoning tracks, so set `CEREBRAS_KEY` if you run wrangle LLM tracks directly.

> Once a plan is learned, IMPRINT can replay it from cache without calling the LLM again.

---

## Project Layout

```
wrangle-imprint/
  README.md              # this file
  CONTRACT.md            # JSON contract between IMPRINT and wrangle
  LICENSE
  .gitignore
  imprint.py             # IMPRINT — LAM + SQLite cache + planner
  wrangle.py             # wrangle — Android controller via ADB + UIAutomator
```

`IMPRINT` expects `WRANGLE_PATH` to point at `wrangle.py` — defaults to the same directory. Rename or move things freely as long as that env var is updated.

---

## Quickstart

### Clone and run checks

```bash
git clone https://github.com/inoahaipro/wrangle-imprint.git
cd wrangle-imprint
mkdir -p ~/.imprint

# Verify Android + ADB + keyboard + DB
python wrangle.py check
python imprint.py check
```

### 1. Basic wrangle test (no LLM required)

With your phone unlocked:

```bash
# Launch Chrome to a URL
python wrangle.py launch_app chrome --url "https://youtube.com"

# Dump the current UI state
python wrangle.py get_state --task "youtube home"
```

You should see JSON with `elements` and a `screen_summary`.

Tap by coordinates:

```bash
python wrangle.py do_action --json '{"action":"tap","x":540,"y":2044}'
```

Need a delay so you can switch to the target app first?

```bash
sh -c 'sleep 5; cd ~/wrangle-imprint/src; python wrangle.py do_action --json "{\"action\":\"tap\",\"x\":540,\"y\":2044}"'
```

### 2. IMPRINT — plan and execute a task

Once OpenClaw is installed and your target session is available in Termux:

```bash
cd ~/wrangle-imprint/src

python imprint.py ask "open youtube.com in chrome and search for lo-fi hip hop then open the first result" --confirmed
```

**First run** — you should see:
- Cache MISS
- LLM call
- Step-by-step execution logs

**Later runs** with a similar intent should:
- Hit the cache
- Reuse the stored plan with **0 tokens**

Inspect stored plans:

```bash
python imprint.py list
python imprint.py stats
```

---

## Contract

The JSON contracts for both communication paths:

- **OpenClaw ↔ IMPRINT**
- **IMPRINT ↔ wrangle**

are documented in [`CONTRACT.md`](./CONTRACT.md), covering:

- Response shapes (`source: cache|llm|replan`, `plan_id`, `tokens`, etc.)
- Action step format (`tap`, `type`, `launch`, `scroll`, `done`, …)
- Result format from wrangle (`ok`, `error`, `screen_hash`, etc.)
- Queue entries, error codes, and state signatures

If you're writing a dispatcher or a new execution adapter, start there.

---

## Roadmap

- [ ] Route wrangle text/browser reasoning through OpenClaw too (so both files share one model/session config)
- [ ] USB-first ADB setup so Wi-Fi isn't required
- [ ] Better browser automation loop for Chrome (via `agent-browser`)
- [ ] Prebuilt skill packs for common tasks: messaging, toggles, settings, etc.

PRs and issues welcome once this is public.

---

## License

MIT License

```
Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

