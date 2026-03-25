# Skill: Debugging This Stack

## WebSocket Won't Connect

### Symptom: `101 Switching Protocols` never received
1. Check Nginx config has `proxy_http_version 1.1` and Upgrade headers
2. Check `docker-compose logs nginx` for upstream errors
3. Verify backend is healthy: `curl http://localhost/healthz`

### Symptom: Connection accepted then immediately closed
1. JWT check failing — test with: `curl -v ws://localhost/ws/rag?token=BAD`
   Expected: close code 4001
2. Check `docker-compose logs backend` for auth exception

### Symptom: Tokens stream then stop mid-way
1. `pause_event` stuck — client may have sent `pause` without `resume`
2. Check task cancellation — did a disconnect cancel the stream task prematurely?
3. OpenAI rate limit — check LangSmith traces for `RateLimitError`

---

## RAG Returns Wrong / Empty Answers

### Symptom: "No relevant documents found"
1. Check collection was ingested: `GET /collections` (add this debug endpoint)
2. Check Chroma persist volume is mounted: `docker-compose exec chroma ls /data/chroma`
3. Embedding model mismatch — collection embedded with different model than query

### Symptom: Answer ignores context
1. Check prompt template — context must be `{context}` not `{documents}`
2. Check `format_docs` function returns numbered passages `[1]...[2]...`
3. Try `k=10` to retrieve more candidates

---

## Docker / uv Issues

### Symptom: `uv sync` fails in Docker with "No solution found"
1. `uv.lock` is out of date — run `uv lock` locally and commit updated lock
2. Python version mismatch — ensure `FROM python:3.12-slim` matches `requires-python = ">=3.12"`

### Symptom: Container restarts in a loop
1. `docker-compose logs backend --tail=50` — look for ImportError or missing env var
2. Missing `.env` key — compare with `.env.example`
3. Chroma not ready yet — check `depends_on` uses `condition: service_healthy`

---

## LangSmith Not Tracing

### Symptom: No runs appearing in LangSmith UI
1. Verify `LANGCHAIN_TRACING_V2=true` in `.env`
2. Verify `LANGCHAIN_API_KEY` is set and valid (not the OpenAI key)
3. Check network: `curl https://api.smith.langchain.com/health` from inside the container
4. Project name: must match `LANGCHAIN_PROJECT` value exactly

---

## Performance

### Symptom: High latency on first query
- Chroma cold start — first query builds the HNSW index. Normal.

### Symptom: Event loop blocking (all WS connections slow simultaneously)
- Synchronous code in async handler
- Run: `uv run python -c "import asyncio; ..."` with `aiomonitor` to inspect tasks
- Common culprits: `requests.*`, `chromadb` sync client, `time.sleep`
