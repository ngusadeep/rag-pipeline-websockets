# Rule: RAG & LangChain

Active whenever editing `backend/rag/` or any file that imports from `langchain`.

## LCEL — Use Pipe Syntax
```python
# ✅ LCEL (modern)
chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# ❌ Legacy (deprecated)
chain = LLMChain(llm=llm, prompt=prompt)
```

## Streaming
- Use `chain.astream(input)` for async token streaming
- Yield each chunk to the WebSocket inside the async for loop
- Always add `await asyncio.sleep(0)` after each `send_json` call

## LangSmith Tracing
```python
from langsmith import traceable

@traceable(name="rag_query", metadata={"version": "1.0"})
async def stream_query(question: str, session_id: str, collection: str):
    ...
```
- Pass `{"session_id": session_id}` in metadata — makes traces filterable
- Set `run_name=session_id` for easy lookup in the LangSmith UI
- NEVER log document content as metadata — only IDs and counts

## Embeddings
- Model: `text-embedding-3-small` — do NOT change to `large` without noting cost impact
- Wrap `embed_documents` with a retry decorator (3 attempts, 2^n backoff, max 30s)
- Cache embeddings for the same document hash to avoid re-embedding on re-ingest

## Chroma
- Always use `persist_directory` from settings — never in-memory for production
- Collection naming: lowercase alphanumeric + underscores only
- `get_or_create_collection` pattern — never assume collection exists

## Retriever
- MMR over cosine similarity to reduce redundant chunks
- `k=6` is the default — expose as a parameter but default to 6
- If retrieval returns 0 docs, yield an error token ("No relevant documents found for your query.") — do NOT let the LLM hallucinate an answer

## Prompt
- Always number context passages: `[1] ... [2] ...`
- System prompt must instruct the model to cite by number
- Keep system prompt under 200 tokens to leave room for context + answer
