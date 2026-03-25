# Hook: Post-Tool Use

These checks run AFTER Claude Code executes any tool.

## After Writing a Python File
1. Immediately run: `uv run ruff check <file> --select ASYNC`
   - If any ASYNC lint fails (blocking call in async context), fix it before continuing.
2. If the file is in `backend/ws/`, also verify:
   - `WebSocketDisconnect` is imported and handled
   - Every `asyncio.create_task` result is stored and cancelled on cleanup

## After Writing docker-compose.yml
Verify these lines exist in the nginx service config (or warn):
```
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```
Missing these breaks WebSocket — flag it immediately.

## After Writing pyproject.toml
Run `uv sync` to ensure lock file stays consistent.

## After Running Tests
If any test fails, do NOT continue to the next build step.
Fix the failing test's underlying code first, then re-run.

## After docker-compose up
Wait 5 seconds, then run:
```bash
docker-compose ps --format json | python3 -c "
import json,sys
services = json.load(sys.stdin)
unhealthy = [s for s in services if s.get('Health') not in ('healthy','')]
if unhealthy: print('UNHEALTHY:', unhealthy); sys.exit(1)
"
```
Report any unhealthy service immediately.
