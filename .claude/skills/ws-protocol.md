# Skill: WebSocket Message Protocol

Complete protocol reference for the RAG WebSocket endpoint.

## Endpoint
```
ws://localhost/ws/rag?token=<jwt>     (dev)
wss://yourdomain.com/ws/rag?token=<jwt>  (prod)
```

## JWT Auth
- Algorithm: HS256
- Payload: `{ "sub": "user_id", "exp": now + 60 }`
- 60-second lifetime — only used for the upgrade handshake
- After connection is established, the token is not re-verified per-message

---

## Client → Server Messages

### query — Start a RAG stream
```json
{
  "type": "query",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "text": "What does the document say about X?",
  "collection": "rag_default"
}
```
- `session_id`: client-generated UUID4, used to correlate all messages
- `collection`: optional, defaults to `rag_default`

### pause — Backpressure signal
```json
{
  "type": "pause",
  "session_id": "550e8400-..."
}
```
Server stops sending tokens. Buffer up to 128 tokens internally.

### resume — Release backpressure
```json
{
  "type": "resume",
  "session_id": "550e8400-...",
  "last_token_idx": 42
}
```
- `last_token_idx`: last token index the client successfully rendered
- Server replays any buffered tokens after this index before continuing

### cancel — Abort generation
```json
{
  "type": "cancel",
  "session_id": "550e8400-..."
}
```
Server cancels the RAG task. No `done` message is sent.

---

## Server → Client Messages

### source — Retrieved documents (sent BEFORE first token)
```json
{
  "type": "source",
  "session_id": "550e8400-...",
  "docs": [
    {"idx": 1, "source": "report.pdf", "page": 3, "score": 0.91},
    {"idx": 2, "source": "notes.txt",  "page": 1, "score": 0.87}
  ]
}
```

### token — Streamed answer chunk
```json
{
  "type": "token",
  "session_id": "550e8400-...",
  "idx": 0,
  "text": "Based on"
}
```
- `idx`: monotonically increasing, used for replay after resume

### done — Stream complete
```json
{
  "type": "done",
  "session_id": "550e8400-...",
  "total_tokens": 312,
  "duration_ms": 4821
}
```

### error — Something went wrong
```json
{
  "type": "error",
  "session_id": "550e8400-...",
  "code": 4010,
  "message": "RAG chain failed: upstream timeout"
}
```
Connection stays OPEN after an error. Client can send another `query`.

---

## Close Codes

| Code | Meaning                  |
|------|--------------------------|
| 1000 | Normal closure           |
| 4001 | Unauthorized / bad JWT   |
| 4002 | Auth timeout (5s)        |
| 4003 | Malformed JSON message   |
| 4010 | RAG chain fatal error    |
| 4020 | Session not found        |
| 4030 | Rate limited             |

---

## Minimal JavaScript Client

```javascript
const token = await getShortLivedJwt();  // your auth endpoint
const ws = new WebSocket(`wss://host/ws/rag?token=${token}`);
const sessionId = crypto.randomUUID();

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  switch (msg.type) {
    case "source": renderSources(msg.docs); break;
    case "token":  appendToken(msg.text); break;
    case "done":   finalise(msg.total_tokens); break;
    case "error":  showError(msg.message); break;
  }
};

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: "query",
    session_id: sessionId,
    text: "What does the document say about X?",
    collection: "rag_default",
  }));
};

// Backpressure example
function pauseIfBufferFull(lastIdx) {
  ws.send(JSON.stringify({ type: "pause", session_id: sessionId }));
  // ... drain render queue ...
  ws.send(JSON.stringify({ type: "resume", session_id: sessionId, last_token_idx: lastIdx }));
}
```
