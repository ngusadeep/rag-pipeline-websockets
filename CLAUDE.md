# CLAUDE.md — RAG Pipeline

> **Read this entire file before writing a single line of code.**
> This is the project brain. Every decision flows from here.
> The `.claude/` folder extends it — agents, commands, hooks, rules, skills are all there.

---

## What We're Building

A production-ready **Retrieval-Augmented Generation (RAG)** pipeline:

| Concern          | Technology                                    |
|------------------|-----------------------------------------------|
| API framework    | FastAPI (Starlette WebSocket, ASGI)           |
| Real-time layer  | WebSockets — bidirectional, token-streaming   |
| RAG framework    | LangChain (LCEL pipe syntax, `astream`)       |
| Embeddings       | OpenAI `text-embedding-3-small`               |
| Vector store     | Chroma (persistent, per-tenant collections)   |
| LLM              | `gpt-4o-mini` via `ChatOpenAI(streaming=True)`|
| Tracing          | LangSmith (`@traceable`, V2 tracing)          |
| Reverse proxy    | Nginx (WebSocket upgrade headers)             |
| Containers       | Docker + Docker Compose                       |
| Package manager  | **uv — never pip**                            |
| Python version   | 3.12+                                         |

---

## Project Structure

```
rag-pipeline/
├── CLAUDE.md                        ← you are here (project brain)
├── .claude/
│   ├── settings.json                ← permissions, env, hook wiring
│   ├── agents/
│   │   ├── ingest-engineer.md       ← owns ingest/ + splitter/embeddings/vectorstore
│   │   ├── rag-chain-engineer.md    ← owns rag/retriever + rag/chain
│   │   ├── ws-engineer.md           ← owns ws/manager + ws/router
│   │   ├── infra-engineer.md        ← owns Dockerfile, compose, nginx
│   │   └── test-engineer.md         ← owns all tests
│   ├── commands/
│   │   ├── build.md                 ← /build  — lint → type-check → compose up
│   │   ├── test.md                  ← /test   — full pytest suite
│   │   ├── lint.md                  ← /lint   — ruff + mypy
│   │   ├── ingest.md                ← /ingest — curl helper
│   │   └── add-dep.md               ← /add-dep — uv add wrapper
│   ├── hooks/
│   │   ├── pre-tool.md              ← secret detection, pip guard, blocking I/O guard
│   │   ├── post-tool.md             ← ruff ASYNC check, nginx verify, health check
│   │   └── on-write.md              ← docstring, type hints, logger, test file rules
│   ├── rules/
│   │   ├── python-async.md          ← uv commands, async patterns, logging, errors
│   │   ├── websocket.md             ← 3 laws, auth, close codes, nginx checklist
│   │   ├── rag-langchain.md         ← LCEL, streaming, LangSmith, embeddings, Chroma
│   │   └── docker-infra.md          ← uv Dockerfile pattern, layer cache, healthchecks
│   └── skills/
│       ├── debugging.md             ← symptom → fix for every common failure
│       ├── uv-workflow.md           ← every uv command you'll need
│       ├── langsmith-tracing.md     ← tracing setup, metadata rules, eval path
│       └── ws-protocol.md           ← full message schema + JS client example
│
├── pyproject.toml                   ← single source of truth for deps (uv)
├── uv.lock                          ← committed, never hand-edited
├── .env.example                     ← committed template
├── .env                             ← gitignored, never committed
├── docker-compose.yml
├── docker-compose.override.yml      ← dev-only (hot reload, bind mounts)
│
├── nginx/
│   └── nginx.conf
│
├── backend/
│   ├── Dockerfile
│   ├── main.py                      ← FastAPI app + lifespan + router includes
│   ├── core/
│   │   ├── config.py                ← pydantic BaseSettings (reads .env)
│   │   └── logging.py               ← structured JSON logger
│   ├── ws/
│   │   ├── codes.py                 ← WS close code enum
│   │   ├── manager.py               ← ConnectionManager with Redis pub/sub
│   │   └── router.py                ← @app.websocket("/ws/rag")
│   ├── rag/
│   │   ├── splitter.py              ← RecursiveCharacterTextSplitter config
│   │   ├── embeddings.py            ← OpenAI embedding wrapper + retry
│   │   ├── vectorstore.py           ← Chroma get_or_create_collection
│   │   ├── retriever.py             ← MMR retriever (k=6, fetch_k=20)
│   │   └── chain.py                 ← LCEL chain, astream, @traceable
│   ├── ingest/
│   │   └── router.py                ← POST /ingest multipart endpoint
│   └── tests/
│       ├── conftest.py              ← shared fixtures (mock OpenAI, Chroma)
│       ├── test_ingest.py
│       ├── test_rag.py
│       └── test_ws.py
│
└── frontend/
    └── index.html                   ← minimal WS test client (vanilla JS)
```

---

## Package Management — uv, Always

```bash
# ✅ Correct
uv sync                          # install from lockfile
uv add httpx                     # add runtime dep
uv add --dev pytest              # add dev dep
uv run pytest                    # run in managed venv
uv run uvicorn backend.main:app  # run server

# ❌ Never
pip install ...
python -m pip ...
```

See `.claude/skills/uv-workflow.md` for the complete reference.

---

## Environment Variables

All secrets in `.env`. Never hard-coded. Never committed.

```bash
# .env.example — commit this
OPENAI_API_KEY=sk-replace-me
LANGCHAIN_API_KEY=ls__replace-me
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=rag-pipeline
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com

CHROMA_PERSIST_DIR=/data/chroma
CHROMA_COLLECTION=rag_default

REDIS_URL=redis://redis:6379/0
JWT_SECRET=change-me-in-production
CORS_ORIGINS=http://localhost:3000
```

`core/config.py` uses `pydantic_settings.BaseSettings`. Any missing key raises at startup — fail fast, never silently.

---

## WebSocket Protocol (Summary)

Full spec: `.claude/skills/ws-protocol.md`

```
ws://host/ws/rag?token=<60s-jwt>

Client → Server:  query | pause | resume | cancel
Server → Client:  source → token(s)... → done  |  error
```

Key behaviours:
- `source` message (retrieved docs) sent **before** first `token`
- `pause`/`resume` for backpressure with 128-token replay buffer
- `cancel` aborts task; no `done` sent
- Connection stays open after `error` — client can retry query

---

## RAG Chain (Summary)

Full spec: `.claude/agents/rag-chain-engineer.md` + `.claude/rules/rag-langchain.md`

```python
chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | ChatOpenAI(model="gpt-4o-mini", streaming=True, temperature=0)
    | StrOutputParser()
)
# async for chunk in chain.astream(question): yield chunk
```

- Splitter: `chunk_size=512`, `chunk_overlap=64`
- Embeddings: `text-embedding-3-small` with retry
- Retriever: MMR, `k=6`, `fetch_k=20`, `lambda_mult=0.5`
- Empty retrieval → error token, never hallucinate

---

## Build Order — Work in This Sequence

Complete and test each step before starting the next. Use `/test` to confirm.

```
Step 1 — Scaffold
  pyproject.toml + uv.lock
  core/config.py  core/logging.py
  backend/main.py (app + healthz endpoint only)
  docker-compose.yml skeleton  nginx/nginx.conf  Dockerfile
  .env.example

Step 2 — Ingest Pipeline  [agent: ingest-engineer]
  rag/splitter.py  rag/embeddings.py  rag/vectorstore.py
  ingest/router.py
  tests/test_ingest.py  →  /test must pass

Step 3 — RAG Chain  [agent: rag-chain-engineer]
  rag/retriever.py  rag/chain.py
  tests/test_rag.py  →  /test must pass

Step 4 — WebSocket Layer  [agent: ws-engineer]
  ws/codes.py  ws/manager.py  ws/router.py
  tests/test_ws.py  →  /test must pass

Step 5 — Infrastructure  [agent: infra-engineer]
  Finalize Dockerfile (uv --frozen)
  Finalize docker-compose.yml (healthchecks, depends_on)
  Finalize nginx/nginx.conf (WS upgrade headers, 3600s timeouts)
  frontend/index.html
  /build  →  all 4 services healthy

Step 6 — Hardening
  Retry decorator on embeddings
  128-token replay buffer in ws/manager.py
  Structured logging in core/logging.py
  /lint  →  zero ruff + mypy errors
  Verify LangSmith traces appear in UI
```

---

## Non-Negotiable Rules

1. **uv only** — never `pip install`, never `python -m pip`
2. **No secrets in source** — all from `.env` via `core/config.py`
3. **No `time.sleep()`** — always `await asyncio.sleep()`
4. **No blocking I/O in async handlers** — no `requests.*`, no sync ORM calls
5. **No bare `except:`** — always specific exception type + log
6. **No silent failures** — every exception logged before re-raise or handling
7. **WebSocket Law** — every WS endpoint has `try/except WebSocketDisconnect` + task cleanup
8. **Nginx Law** — `proxy_http_version 1.1` + Upgrade headers always present
9. **Type hints everywhere** — every function signature, checked by mypy strict
10. **Tests before next step** — `/test` passes 100% before advancing in build order
