# Command: /test

Runs the full test suite. Use after any code change.

## Steps

```bash
uv run pytest backend/tests/ --tb=short -q --asyncio-mode=auto
```

## Flags
- `--tb=short` — compact tracebacks
- `-q` — quiet output, just pass/fail counts
- `--asyncio-mode=auto` — no need to decorate every async test

## Coverage (optional)
```bash
uv run pytest backend/tests/ --cov=backend --cov-report=term-missing -q
```

## What Must Pass
- All tests in `test_ingest.py`, `test_rag.py`, `test_ws.py`
- Zero warnings treated as errors (configured in pyproject.toml)
- No calls to real OpenAI or Chroma APIs (mocked)

## On Failure
Report the failing test name, the exact assertion error, and the file + line number.
Then fix the code (not the test) unless the test itself is wrong.
