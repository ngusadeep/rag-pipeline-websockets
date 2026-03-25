# Rule: Python & Async

Always active for all Python files in this project.

## Package Management — uv ONLY
```bash
# Install deps
uv sync

# Add a package
uv add httpx

# Add dev package
uv add --dev pytest

# Run a script
uv run python script.py

# Run a tool
uv run pytest
uv run ruff check .
uv run mypy .

# NEVER use:
pip install ...       ❌
python -m pip ...     ❌
pipenv install ...    ❌
poetry add ...        ❌
```

## Async Rules
- All functions that touch I/O (DB, Redis, HTTP, filesystem) must be `async def`
- `await asyncio.sleep(0)` inside any tight loop to yield to the event loop
- Use `asyncio.gather()` for concurrent independent I/O, not sequential `await`
- Use `asyncio.wait_for(coro, timeout=N)` for any external call that might hang
- Background tasks: always `asyncio.create_task()`, always cancel on cleanup

## Python Version
Minimum Python 3.12. Use modern syntax:
- `X | Y` union types (not `Optional[X]` or `Union[X, Y]`)
- `match` statements for message type dispatch
- `f"{x!r}"` for repr in log messages
- `@dataclass(slots=True)` for simple data holders

## Logging
```python
import logging
logger = logging.getLogger(__name__)
# Use:
logger.info("message %s", var)   # ✅ lazy formatting
logger.info(f"message {var}")    # ❌ eager formatting (wastes CPU if INFO disabled)
```

## Error Handling
```python
# ✅ Always specific, always logged
try:
    result = await some_call()
except httpx.TimeoutException as e:
    logger.warning("Timeout calling %s: %s", url, e)
    raise

# ❌ Never bare except
try:
    ...
except:        # catches KeyboardInterrupt too!
    pass
```
