# Command: /lint

Runs the full linting and type-checking pipeline.

## Steps

```bash
# Ruff: lint + auto-fix safe issues
uv run ruff check backend/ --fix

# Ruff: format
uv run ruff format backend/

# Mypy: strict type check
uv run mypy backend/ --ignore-missing-imports --strict
```

## Rules Enforced by Ruff
- `E` / `F` — standard pyflakes/pycodestyle
- `I` — import sorting (isort-compatible)
- `UP` — pyupgrade (modern Python idioms)
- `B` — bugbear (common Python bugs)
- `ASYNC` — async-specific lints (no `time.sleep`, blocking calls in async)

## Zero Tolerance
Claude Code must fix ALL ruff and mypy errors before marking any task complete.
Never add `# type: ignore` without a comment explaining why.
Never add `# noqa` without a comment explaining why.
