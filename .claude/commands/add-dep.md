# Command: /add-dep [package]

Adds a Python dependency using uv. Never use pip.

## Steps
```bash
# Add runtime dependency
uv add $ARG1

# Add dev-only dependency
uv add --dev $ARG1

# After adding, always sync
uv sync
```

## Rules
- Always use `uv add` — never `pip install`
- Always commit both `pyproject.toml` AND `uv.lock`
- Dev tools (pytest, ruff, mypy) go under `[tool.uv.dev-dependencies]`
- After adding, re-run `/lint` and `/test` to confirm nothing broke

## Pin Versions
When adding, prefer a minimum-version pin:
```bash
uv add "langchain>=0.3,<0.4"
```
Do not use unpinned `uv add langchain` for core dependencies.
