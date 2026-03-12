# wrangle-imprint v2.0.0

`wrangle-imprint` is a local Android automation stack made of two Python files:

- **`wrangle.py`**: executes phone actions over ADB and exposes ranked UI state from UIAutomator.
- **`imprint.py`**: a Local Agentic Model (LAM) that learns reusable plans, stores them in SQLite, and replays them with zero tokens on cache hits.

The core behavior is: **LLM teaches once, IMPRINT replays forever**.

## Architecture

```text
OpenClaw chat / dispatcher
          │
          ▼
      imprint.py  <->  ~/.imprint/memory.db
          │
          ▼
      wrangle.py  <->  adb localhost:${ADB_PORT}
          │
          ▼
      Android device
```

See the machine-readable interface contract in [`CONTRACT.md`](./CONTRACT.md).

## Requirements

- Python 3.11+
- `requests` (`pip install -r requirements.txt`)
- ADB reachable at `localhost:${ADB_PORT}` (default port in code: **45171**)
- ADBKeyboard installed + active (`com.android.adbkeyboard/.AdbIME`)
- OpenClaw CLI configured (required for new-plan generation in `imprint.py`)
- Optional: `CEREBRAS_KEY` for direct `wrangle.py` browser/text tracks

## Environment variables

### Shared / execution

- `ADB_PORT` (default `45171`)
- `DEVICE_WIDTH` (default `1080`)
- `DEVICE_HEIGHT` (default `2340`)

### IMPRINT (`imprint.py`)

- `OPENCLAW_SESSION` (default `main`)
- `IMPRINT_DB` (default `~/.imprint/memory.db`)
- `IMPRINT_THRESHOLD` (default `0.72`)
- `IMPRINT_CONFIRM` (default `2`)
- `IMPRINT_MAX_STEPS` (default `20`)
- `IMPRINT_TIMEOUT` (default `120`)
- `IMPRINT_RETRIES` (default `2`)
- `IMPRINT_DEBUG` (`1` to enable debug logging)
- `WRANGLE_PATH` (optional path override)

### wrangle (`wrangle.py`)

- `CEREBRAS_KEY` (optional for direct browser/text tracks)
- `WRANGLE_MAX_UI_ELEMENTS` (default `24`)
- `WRANGLE_ADB_RETRIES` (default `3`)
- `OLLAMA_URL` (default `http://localhost:11434`)
- `VISION_MODEL` (default `moondream`)

## Quickstart

```bash
pip install -r requirements.txt
mkdir -p ~/.imprint

python wrangle.py check
python imprint.py check
```

Learn and execute a task:

```bash
python imprint.py ask "open youtube and search for lo-fi hip hop"
```

Useful commands:

```bash
python imprint.py ask "turn off wifi" --dry
python imprint.py ask "delete that message" --confirmed
python imprint.py ask "open settings" --queue
python imprint.py flush
python imprint.py list
python imprint.py plans-json
python imprint.py stats
python imprint.py apps

python wrangle.py get_state --task "current screen"
python wrangle.py do_action --json '{"action":"tap","x":540,"y":1200}'
python wrangle.py list_apps
python wrangle.py launch_app chrome --url "https://youtube.com"
```

## Safety / reliability features in v2

- Parameterized plan templates (`{contact}`, `{message}`, `{app}`, `{query}`)
- Element-based targeting (text / id / content-desc)
- Per-step retry + structured error reporting
- Mid-task replan through OpenClaw when execution drifts
- Screen-change verification and loop detection
- Queue/flush mode for temporary ADB outages
- Destructive-action guard unless `--confirmed` is passed

## License

MIT. See [`LICENSE`](./LICENSE).
