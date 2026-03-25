# Hook: Pre-Tool Use

These checks run BEFORE Claude Code executes any tool. Fail fast if violated.

## Secret Detection
Before writing ANY file, scan the content for:
- Strings matching `sk-[a-zA-Z0-9]{40,}` (OpenAI keys)
- Strings matching `ls__[a-zA-Z0-9]{20,}` (LangSmith keys)
- Any string that looks like an API key, password, or token hardcoded in source

If detected: **STOP. Do not write the file.** Report the line and ask the user to use `.env` instead.

## pip Guard (HARD BLOCK)
If any Bash command contains any of the following: **STOP immediately.**
- `pip install`
- `pip3 install`
- `python -m pip`
- `pipenv install`
- `poetry add`

Replace with the correct uv command:
| Blocked | Use instead |
|---------|-------------|
| `pip install X` | `uv add X` |
| `pip install -r requirements.txt` | `uv sync` |
| `pip install --dev X` | `uv add --dev X` |

## Formatter Guard
If any Bash command runs `ruff format` — **STOP.**
Black owns formatting in this project. Use `uv run black backend/` instead.
Ruff is lint-only here (`ruff check`, never `ruff format`).

## Blocking I/O Guard
Before writing to `backend/ws/` or any `async def`, scan for:
- `time.sleep(` → replace with `await asyncio.sleep(`
- `requests.get(` / `requests.post(` → replace with `httpx.AsyncClient`
- Synchronous file I/O inside async functions

If found: **STOP.** Fix the violation before proceeding.

## Overwrite Guard
Before overwriting an existing file, read it first.
If content differs, show a brief diff summary and confirm before overwriting.
(Exception: files touched by `black` or `ruff check --fix`.)
