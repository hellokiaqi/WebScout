# WebScout A2A Protocol Server (`Manus` Agent)

English | [简体中文](README_zh.md)

This module provides an experimental HTTP integration of Google's [Agent2Agent (A2A) protocol](https://google.github.io/A2A/#/documentation) wrapping WebScout's general-purpose `Manus` agent (`A2AManus`).

> Current status: Supports synchronous JSON-RPC `message/send` requests (`streaming = False`, `pushNotifications = True`). Each incoming task creates an `A2AManus` instance (`max_steps = 3`) via `ManusExecutor`.

---

## Key Files

- `protocol/a2a/app/main.py`: Starlette + Uvicorn server entrypoint (`--host`, `--port`), `AgentCard` registration, and skill metadata (`Python Execute`, `Browser use`, `Replace String`, `Ask human`, `terminate`).
- `protocol/a2a/app/agent.py`: `A2AManus(Manus)` subclass implementing `invoke()` for `text` / `text/plain` inputs (raises `NotImplementedError` on `stream()`).
- `protocol/a2a/app/agent_executor.py`: `ManusExecutor(AgentExecutor)` bridging A2A `RequestContext` and `EventQueue` to `A2AManus.invoke()`.

---

## Prerequisites

1. Activate your WebScout Python 3.12 virtual environment with root dependencies installed (`uv pip install -r requirements.txt`) and configure `config/config.toml`.
2. Install the A2A SDK and `python-dotenv` (imported by `protocol/a2a/app/main.py`):

```bash
uv pip install a2a-sdk==0.2.5 python-dotenv
```

---

## Starting the Server

Run from the WebScout repository root:

```bash
python -m protocol.a2a.app.main --host localhost --port 10000
```

| CLI Option | Default | Description |
| --- | --- | --- |
| `--host` | `localhost` | Bind host address (also used to populate `AgentCard.url`). |
| `--port` | `10000` | Bind TCP port. |

---

## Calling the A2A Endpoints

### 1. Fetch the Agent Card

```bash
curl http://localhost:10000/.well-known/agent.json
```

Example response excerpt:

```json
{
  "capabilities": {
    "pushNotifications": true,
    "streaming": false
  },
  "defaultInputModes": ["text", "text/plain"],
  "defaultOutputModes": ["text", "text/plain"],
  "description": "A versatile agent that can solve various tasks using multiple tools including MCP-based tools",
  "name": "Manus Agent",
  "url": "http://localhost:10000/",
  "version": "1.0.0"
}
```

### 2. Send a Task via JSON-RPC (`message/send`)

```bash
curl --location 'http://localhost:10000/' \
  --header 'Content-Type: application/json' \
  --data '{
    "id": 1,
    "jsonrpc": "2.0",
    "method": "message/send",
    "params": {
      "message": {
        "messageId": "msg-001",
        "role": "user",
        "parts": [{"text": "Calculate the first 10 Fibonacci numbers using Python and print them."}]
      }
    }
  }'
```

### 3. Using the Official A2A Sample Client (Optional)

You can also connect the official A2A CLI or Web UI host from [`google-a2a/a2a-samples`](https://github.com/google-a2a/a2a-samples) to `http://localhost:10000`:

```bash
git clone https://github.com/google-a2a/a2a-samples.git
cd a2a-samples/samples/python/hosts/cli
uv run . --agent http://localhost:10000
```

---

## References

- [A2A Protocol Documentation](https://google.github.io/A2A/#/documentation)
