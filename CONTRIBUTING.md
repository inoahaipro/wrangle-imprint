# Contributing to wrangle-imprint

Thanks for contributing.

## Development setup

```bash
git clone https://github.com/inoahaipro/wrangle-imprint.git
cd wrangle-imprint
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Validate before opening a PR

```bash
python -m py_compile wrangle.py imprint.py
python wrangle.py check
python imprint.py check
```

If you changed docs or the static site, keep `README.md`, `CONTRACT.md`, and `index.html` aligned with current behavior.

## Pull requests

- Use clear, scoped commits.
- Explain user-visible behavior changes and any contract changes.
- Include command output for checks you ran.
