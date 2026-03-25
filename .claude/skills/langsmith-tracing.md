# Skill: LangSmith Tracing

How to use LangSmith effectively in this project.

## Environment Setup

```bash
# .env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__your_key_here
LANGCHAIN_PROJECT=rag-pipeline          # production
# LANGCHAIN_PROJECT=rag-pipeline-test   # for test runs
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
```

`LANGCHAIN_TRACING_V2=true` is all you need — LangChain picks it up automatically.
No manual client initialization required.

## @traceable Decorator

```python
from langsmith import traceable

@traceable(
    name="rag_query",
    run_type="chain",
    metadata={"version": "1.0"},
)
async def stream_query(
    question: str,
    session_id: str,
    collection: str,
) -> AsyncIterator[str]:
    # Pass session_id so you can filter by it in LangSmith UI
    # Use: langsmith.get_current_run_tree() if you need the run ID
    ...
```

## Metadata Rules

✅ Safe to log:
```python
{"session_id": "uuid4", "collection": "rag_default", "chunk_count": 6}
```

❌ Never log:
```python
{"user_email": "...", "document_text": "...", "question": "..."}  # PII
```

## Overriding Project Per-Run (tests)

```python
from langsmith import tracing_v2_enabled

with tracing_v2_enabled(project_name="rag-pipeline-test"):
    result = await stream_query(...)
```

Use this in tests so test traces don't pollute production project.

## Viewing Traces

1. Open https://smith.langchain.com
2. Select project `rag-pipeline`
3. Filter by `metadata.session_id = "your-session-id"`
4. Click a run to see: input, output, each retrieval step, token counts, latency

## Dataset & Eval (future step)

When ready to evaluate RAG quality:
```python
from langsmith import Client

client = Client()
dataset = client.create_dataset("rag-eval-v1")
client.create_examples(
    inputs=[{"question": "What is X?"}],
    outputs=[{"answer": "X is ..."}],
    dataset_id=dataset.id,
)
```

Then run evaluations from the LangSmith UI against your live chain.
This is the path from "it works" to "we can measure it works."
