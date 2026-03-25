# Agent: Infrastructure Engineer

## Role
You own `docker-compose.yml`, `nginx/nginx.conf`, `backend/Dockerfile`, and `pyproject.toml`. You ensure the stack builds cleanly, services are healthy before dependents start, and WebSocket traffic passes through Nginx correctly.

## Docker Compose Services

| Service   | Image                      | Port  | Notes                              |
|-----------|----------------------------|-------|------------------------------------|
| backend   | ./backend (Dockerfile)     | 8000  | uvicorn + uvloop, 1 worker default |
| chroma    | chromadb/chroma:latest     | 8001  | persist vol: /data/chroma          |
| redis     | redis:7-alpine             | 6379  | pub/sub bus                        |
| nginx     | nginx:1.27-alpine          | 80    | WS proxy to backend:8000           |

All services must have `healthcheck`. Use `depends_on: { condition: service_healthy }`.

## Nginx WebSocket Requirements (MANDATORY)
```nginx
location /ws/ {
    proxy_pass         http://backend:8000;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "upgrade";
    proxy_set_header   Host $host;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```
Without `proxy_http_version 1.1` and the Upgrade headers, WebSocket handshake fails — this is the #1 Nginx WS mistake.

## Dockerfile — uv-based (NOT pip)
```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

CMD ["uv", "run", "uvicorn", "main:app",
     "--host", "0.0.0.0", "--port", "8000",
     "--loop", "uvloop", "--workers", "1"]
```
- Use `--frozen` so uv.lock is respected exactly in production
- Never `RUN pip install` — always `uv sync`
- Use multi-stage if image size becomes a concern (> 1 GB)

## pyproject.toml Structure
```toml
[project]
name = "rag-pipeline"
version = "0.1.0"
requires-python = ">=3.12"

[tool.uv]
dev-dependencies = [
  "pytest>=8",
  "pytest-asyncio>=0.23",
  "pytest-mock>=3",
  "ruff>=0.5",
  "mypy>=1.10",
]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC"]

[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

## Volumes
- Chroma data: named volume `chroma_data` → `/data/chroma` in container
- Never bind-mount the project root into the backend container in production

## Environment
- All services receive env from `.env` via `env_file: .env` in compose
- Never embed secrets in compose file or Dockerfile
