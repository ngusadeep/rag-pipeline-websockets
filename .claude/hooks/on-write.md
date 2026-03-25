# Hook: On File Write

Enforced on EVERY file write, no exceptions.

## Python Files
- First line of every module: module-level docstring (one sentence minimum)
- Every `async def` that touches external I/O must have a return type annotation
- No bare `except:` — always `except SomeException as e:` and log `e`
- `logger = logging.getLogger(__name__)` in every module that logs — never `print()`

## WebSocket Files (backend/ws/)
- `WebSocketDisconnect` must be imported from `fastapi`
- Every `asyncio.create_task(...)` result must be assigned to a variable
- Every task variable must appear in a `finally:` or `except WebSocketDisconnect:` block with `.cancel()`

## Environment / Config
- `.env` must NEVER be written — it is gitignored
- `.env.example` must be updated whenever a new env var is used in `core/config.py`

## Tests
- Every new module in `backend/` must have a corresponding test file created in `backend/tests/`
- Test file name: `test_<module_name>.py`
- Minimum: one happy-path and one unhappy-path test per public function
