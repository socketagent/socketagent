# SocketAgent

Minimal API discovery for LLM agents. SocketAgent lets servers expose a tiny, consistent descriptor that clients can fetch once to understand available endpoints, schemas, and examples — without maintaining heavyweight specs.

## Overview

- Server: Adds lightweight annotations and exposes `/.well-known/socket-agent` (JSON)
- Client: Fetches the descriptor, plans calls, and interacts with endpoints
- Protocol: Small, stable JSON shape designed for agents (3–8KB typical)

## How It Works

- Annotate FastAPI endpoints with `@socket.describe(...)` to add summary, optional request/response JSON Schemas, and example commands.
- Add `SocketAgentMiddleware(app, name=..., description=..., base_url=optional)`; it registers `GET /.well-known/socket-agent` and builds the descriptor by inspecting your routes.
- The descriptor is cached on first request. If `base_url` is not provided, it’s inferred from the incoming request’s scheme and host.

## Server Integration (FastAPI)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from socket_agent import socket, SocketAgentMiddleware

app = FastAPI(title="Todo API")

class TodoCreate(BaseModel):
    text: str

@app.post("/todo")
@socket.describe(
    "Create a new todo item",
    request_schema={
        "type": "object",
        "properties": {"text": {"type": "string"}},
        "required": ["text"],
    },
    response_schema={
        "type": "object",
        "properties": {"id": {"type": "string"}, "text": {"type": "string"}},
    },
    examples=['curl -X POST /todo -d "{\"text\":\"buy milk\"}"'],
)
async def create_todo(todo: TodoCreate):
    return {"id": "1", "text": todo.text}

SocketAgentMiddleware(
    app,
    name="Todo API",
    description="Simple todo list management API",
)
```

Then fetch the descriptor:

```bash
curl http://localhost:8000/.well-known/socket-agent
```

## Descriptor Protocol

Path: `GET /.well-known/socket-agent`

Shape (fields are stable; unknown fields may be ignored by clients):

```json
{
  "name": "API name",
  "description": "API description",
  "base_url": "https://api.example.com",
  "endpoints": [
    { "path": "/todo", "method": "POST", "summary": "Create a todo" }
  ],
  "schemas": {
    "/todo": {
      "request": { "type": "object", "properties": {"text": {"type":"string"}}, "required": ["text"] },
      "response": { "type": "object", "properties": {"id": {"type":"string"}, "text": {"type":"string"}} }
    }
  },
  "auth": { "type": "none", "description": null },
  "examples": [
    "curl -X POST /todo -d '{\"text\":\"buy milk\"}'"
  ],
  "ui": null,
  "specVersion": "2025-01-01"
}
```

Notes:
- `endpoints` lists each exposed route and HTTP method with a short summary.
- `schemas` are optional per-path JSON Schemas for request/response bodies.
- `auth` currently defaults to `{ type: "none" }`; servers can document other schemes.
- `examples` are free-form, usually curl examples to guide calling patterns.
- `ui` is optional UI hinting that clients may ignore.
- Size guidance: recommended under 3KB; hard limit 8KB (server warns/raises).

## Client Responsibilities

- Discover: GET `/.well-known/socket-agent` and cache the descriptor.
- Plan: Use `base_url`, `endpoints`, `schemas`, and `examples` to decide calls.
- Call: Execute HTTP requests to the listed `path` with the specified `method`.
- Auth: Honor `auth` if present; default is none.
- UX: Optionally render simple forms using `schemas` and `ui` hints.

Example (pseudo-code):

```python
import httpx, json

async def agent_flow(base):
    async with httpx.AsyncClient() as client:
        desc = (await client.get(f"{base}/.well-known/socket-agent")).json()
        todo = {"text": "buy milk"}
        res = await client.post(f"{base}/todo", json=todo)
        return res.json()
```

## Design Philosophy

- Minimal over maximal: A tiny, stable shape that agents can learn quickly.
- Dumb server, smart client: The server only describes; clients discover, plan, and optimize.
- Progressive enhancement: Start with summaries; add schemas/examples where helpful.

## Limits and Behavior

- Descriptor size: warning > 3KB, error > 8KB.
- Route filtering: `/.well-known/...` is excluded from `endpoints`.
- Caching: Descriptor is built once per app process and reused.

## License

MIT

