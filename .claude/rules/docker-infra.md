# Rule: Docker & Infrastructure

Active whenever editing `docker-compose.yml`, `nginx/`, or `backend/Dockerfile`.

## uv in Docker — Canonical Pattern
```dockerfile
FROM python:3.12-slim

# Copy uv binary from official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copy lockfiles first (layer cache)
COPY pyproject.toml uv.lock ./

# Install with frozen lock — never resolves, always reproducible
RUN uv sync --frozen --no-dev

# Copy source
COPY backend/ ./backend/

CMD ["uv", "run", "uvicorn", "backend.main:app", \
     "--host", "0.0.0.0", "--port", "8000", \
     "--loop", "uvloop", "--workers", "1"]
```

## Layer Cache Order (ALWAYS copy lockfiles before source)
```
COPY pyproject.toml uv.lock ./   ← changes rarely → cached
RUN uv sync --frozen             ← cached until ^^ changes
COPY . .                         ← changes often → not cached
```
Reversing this order makes every build reinstall all packages. Do not do it.

## Health Checks
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 15s
```
Every service needs one. `depends_on: { condition: service_healthy }` requires it.

## Nginx Checklist
When writing `nginx/nginx.conf`, verify ALL of these are present:
- [ ] `proxy_http_version 1.1;`
- [ ] `proxy_set_header Upgrade $http_upgrade;`
- [ ] `proxy_set_header Connection "upgrade";`
- [ ] `proxy_read_timeout 3600s;`
- [ ] `proxy_send_timeout 3600s;`
- [ ] `proxy_set_header Host $host;`
- [ ] `proxy_set_header X-Real-IP $remote_addr;`

## Volumes
- Named volumes only for persistent data (chroma, redis)
- Never bind-mount the project root in production (`./:/app` anti-pattern)
- Dev override is acceptable in `docker-compose.override.yml` only

## Secrets
- All secrets via `env_file: .env` in compose
- Never `environment: OPENAI_API_KEY=sk-...` in compose file
- `.env` is gitignored — `.env.example` is committed with placeholder values
