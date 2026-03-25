# Command: /ingest [file] [collection?]

Ingests a document into the RAG pipeline via the REST endpoint.

## Usage
```
/ingest ./docs/report.pdf
/ingest ./docs/notes.txt my_collection
```

## Steps
```bash
# Default collection
curl -X POST http://localhost/ingest \
  -F "file=@$ARG1" \
  -F "collection=${ARG2:-rag_default}"
```

## Expected Response
```json
{
  "status": "ok",
  "chunks": 42,
  "collection": "rag_default",
  "doc_id": "uuid4-here"
}
```

## On Error
- `413` → file too large (> 50 MB)
- `415` → unsupported file type (only PDF, TXT, MD)
- `422` → validation error (check field names)
- `500` → check `docker-compose logs backend` for traceback
