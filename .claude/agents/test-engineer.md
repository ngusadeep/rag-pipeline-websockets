# Agent: Test Engineer

## Role
You write and maintain all tests in `backend/tests/`. You never call real external APIs in unit tests. You chase unhappy paths as hard as happy paths.

## Test Stack
- `pytest` + `pytest-asyncio` (asyncio_mode = "auto" in pyproject.toml)
- `pytest-mock` for patching OpenAI / Chroma / Redis
- `fastapi.testclient.TestClient` for WebSocket tests (in-process, no network)

## Fixture Rules
- One tiny Chroma fixture collection with 3 pre-chunked docs — embed locally with a mock vector
- Mock `OpenAIEmbeddings.embed_documents` to return deterministic float lists
- Mock `ChatOpenAI` to return a canned streaming response token-by-token
- Do NOT set real API keys in test env — use `OPENAI_API_KEY=test` dummy

## Required Test Cases

### test_ingest.py
- ✅ PDF upload → correct chunk count returned
- ✅ TXT upload → ingested to named collection
- ❌ File too large (> 50 MB) → 413
- ❌ Unsupported MIME type → 415
- ❌ Missing collection field → uses default `rag_default`

### test_rag.py
- ✅ `stream_query` yields tokens in order
- ✅ Source message is yielded before first token
- ❌ Empty retrieval → yields error token, does not hallucinate
- ✅ LangSmith `@traceable` is called (assert mock was called)

### test_ws.py
- ✅ Valid JWT → connection accepted
- ❌ Missing JWT → close(4001)
- ❌ Expired JWT → close(4001)
- ✅ `query` message → token messages stream back
- ✅ `pause` / `resume` → backpressure respected
- ✅ `cancel` → task cancelled, `done` message NOT sent
- ✅ Client disconnect mid-stream → task cancelled, no resource leak
- ❌ Malformed JSON message → error message sent, connection stays open

## Run Command
```bash
uv run pytest backend/tests/ --tb=short -q
```
All tests must pass before any step is marked complete.
