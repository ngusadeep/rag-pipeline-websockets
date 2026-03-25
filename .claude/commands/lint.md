# Command: /lint

Runs the full linting and type-checking pipeline (Option A: ruff + black + mypy).

## Steps

```bash
# 1. Ruff: lint + auto-fix safe issues (NO formatting — black owns that)
uv run ruff check backend/ --fix

# 2. Black: formatting
uv run black backend/

# 3. Mypy: strict type check
uv run mypy backend/ --ignore-missing-imports --strict
```

## Who Does What
- **ruff** — finds bugs, bad imports, outdated syntax, async violations
- **black** — owns all formatting (line length, quotes, spacing)
- **mypy** — catches type errors

They do not overlap. ruff has `E501` ignored so black controls line length.

## Rules Enforced by Ruff
- `E` / `F` — pyflakes/pycodestyle (real bugs + style)
- `I` — import sorting
- `UP` — modern Python idioms (`str | None` not `Optional[str]`)
- `B` — bugbear (bare `except:`, mutable defaults)
- `ASYNC` — async-specific: catches `time.sleep()` and blocking I/O in async functions

## Zero Tolerance
Fix ALL ruff and mypy errors before marking any task complete.
Never add `# type: ignore` without a comment explaining why.
Never add `# noqa` without a comment explaining why.
