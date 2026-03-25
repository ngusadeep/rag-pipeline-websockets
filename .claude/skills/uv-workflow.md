# Skill: uv Workflow

Quick reference for every uv operation used in this project.
Never use pip, pipenv, or poetry. Always use uv.

## Daily Commands

```bash
# Install / sync all deps from lockfile
uv sync

# Add a runtime dependency (updates pyproject.toml + uv.lock)
uv add httpx
uv add "langchain>=0.3,<0.4"

# Add a dev-only dependency
uv add --dev pytest pytest-asyncio pytest-mock ruff mypy

# Remove a dependency
uv remove httpx

# Upgrade a single package
uv add "httpx>=0.28"

# Upgrade all packages (careful — review diff before committing)
uv lock --upgrade

# Run any command in the managed venv
uv run python script.py
uv run pytest
uv run uvicorn backend.main:app --reload
uv run ruff check .
uv run mypy backend/

# Run a one-off tool without adding it to the project
uvx ruff check .
uvx mypy backend/
```

## In Docker

```dockerfile
# Copy uv binary
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Install locked deps — no network resolution, fully reproducible
RUN uv sync --frozen --no-dev

# Run the app
CMD ["uv", "run", "uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## pyproject.toml Layout

```toml
[project]
name = "rag-pipeline"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.115",
  "uvicorn[standard]>=0.30",
  "uvloop>=0.20",
  "langchain>=0.3,<0.4",
  "langchain-openai>=0.2",
  "langchain-chroma>=0.1",
  "langsmith>=0.1",
  "chromadb>=0.5",
  "redis[asyncio]>=5",
  "pydantic-settings>=2",
  "python-jose[cryptography]>=3",
  "python-multipart>=0.0.9",
  "pypdf>=4",
  "httpx>=0.27",
]

[tool.uv]
dev-dependencies = [
  "pytest>=8",
  "pytest-asyncio>=0.23",
  "pytest-mock>=3",
  "ruff>=0.5",
  "black>=24",
  "mypy>=1.10",
  "types-python-jose",
]

[tool.black]
line-length = 88
target-version = ["py312"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC"]
ignore = ["E501"]   # black controls line length

[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["backend/tests"]
```

## Committing Dependencies

Always commit BOTH files together:
```bash
git add pyproject.toml uv.lock
git commit -m "chore: add httpx dependency"
```

Committing only `pyproject.toml` without `uv.lock` means the next `uv sync` may resolve different versions.
`uv.lock` is the source of truth for reproducible builds.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `No solution found` | `uv lock --upgrade` or relax version constraint |
| `Lockfile is out of date` | `uv sync` (it will update the lock) |
| Import works locally, fails in Docker | Check `--no-dev` not stripping needed dep |
| Slow Docker builds | Ensure `COPY pyproject.toml uv.lock ./` before `COPY . .` |
