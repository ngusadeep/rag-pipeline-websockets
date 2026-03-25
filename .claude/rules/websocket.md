# Rule: WebSocket

Active whenever editing `backend/ws/` or any file that imports from `fastapi.websockets`.

## The Three Laws of WebSocket Handlers

### Law 1 — Always catch WebSocketDisconnect
```python
# ✅ Correct
@app.websocket("/ws/rag")
async def ws_rag(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            msg = await websocket.receive_json()
            ...
    except WebSocketDisconnect:
        await cleanup(websocket)

# ❌ Wrong — disconnect propagates as unhandled exception
@app.websocket("/ws/rag")
async def ws_rag(websocket: WebSocket):
    await websocket.accept()
    while True:
        msg = await websocket.receive_json()
```

### Law 2 — Always cancel background tasks on disconnect
```python
task = asyncio.create_task(stream_rag(...))
try:
    ...
except WebSocketDisconnect:
    task.cancel()          # ✅ Always
    await manager.disconnect(websocket)
```

### Law 3 — Never block the event loop
```python
# ✅
await asyncio.sleep(0)     # yield point inside generator loop

# ❌
time.sleep(0.1)            # blocks ALL connections on this worker
```

## Auth Pattern (query param JWT)
```python
@app.websocket("/ws/rag")
async def ws_rag(
    websocket: WebSocket,
    token: str = Query(None),
):
    payload = verify_jwt(token)          # raises if invalid
    if payload is None:
        await websocket.close(code=4001, reason="Unauthorized")
        return
    await websocket.accept()
    ...
```
Token lifetime: 60 seconds. Short-lived so log exposure is minimal.

## Close Codes (defined in ws/codes.py)
| Code | Meaning                     |
|------|-----------------------------|
| 4001 | Unauthorized / bad JWT      |
| 4002 | Auth timeout                |
| 4003 | Malformed message           |
| 4010 | RAG chain error             |
| 4020 | Session not found           |
| 4030 | Rate limited                |

## Nginx Requirements
Without these, WebSocket upgrade silently fails:
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```
