# Command: /build

Builds and starts the full stack. Use this to verify everything compiles and services are healthy.

## Steps

```bash
# 1. Sync dependencies
uv sync

# 2. Lint check (must pass before build)
uv run ruff check backend/

# 3. Type check
uv run mypy backend/ --ignore-missing-imports

# 4. Build and start containers
docker-compose build --no-cache
docker-compose up -d

# 5. Wait for health
docker-compose ps

# 6. Smoke test: check backend is reachable
curl -sf http://localhost/healthz || echo "BACKEND NOT HEALTHY"
```

## Expected Output
All 4 services show `healthy` in `docker-compose ps`. `/healthz` returns `{"status":"ok"}`.

## On Failure
- Check `docker-compose logs backend` for Python errors
- Check `docker-compose logs nginx` for proxy errors
- Verify `.env` has all required keys (compare with `.env.example`)
