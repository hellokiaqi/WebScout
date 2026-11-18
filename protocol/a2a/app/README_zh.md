# WebScout A2A 协议服务端（`Manus` Agent）

[English](README.md) | 简体中文

本模块是 Google [Agent2Agent (A2A) 协议](https://google.github.io/A2A/#/documentation) 的实验性 HTTP 服务端适配，将 WebScout 底层的通用多工具 `Manus` 智能体封装为 `A2AManus` 对外提供服务。

> 当前状态：仅支持非流式 JSON-RPC `message/send` 调用（`streaming = False`，`pushNotifications = True`）。每次请求由 `ManusExecutor` 创建 `A2AManus` 实例（默认 `max_steps = 3`）执行并返回文本产物。

---

## 核心文件说明

- `protocol/a2a/app/main.py`：基于 Starlette + Uvicorn 的服务启动入口（支持 `--host`、`--port`），负责构建 `AgentCard` 并注册 5 项技能描述（`Python Execute`、`Browser use`、`Replace String`、`Ask human`、`terminate`）。
- `protocol/a2a/app/agent.py`：`A2AManus(Manus)` 子类，实现 `invoke()` 接口并声明 `SUPPORTED_CONTENT_TYPES = ["text", "text/plain"]`（`stream()` 会抛出 `NotImplementedError`）。
- `protocol/a2a/app/agent_executor.py`：`ManusExecutor(AgentExecutor)`，负责从 A2A `RequestContext` 读取用户输入、调用 `A2AManus.invoke()`，并将结果封装为 `Task` 产物写入 `EventQueue`。

---

## 环境准备

1. 在 WebScout 仓库根目录激活 Python 3.12 虚拟环境，安装主依赖（`uv pip install -r requirements.txt`）并配置好 `config/config.toml`。
2. 额外安装 A2A SDK 与 `python-dotenv`（`protocol/a2a/app/main.py` 启动时调用 `load_dotenv()`）：

```bash
uv pip install a2a-sdk==0.2.5 python-dotenv
```

---

## 启动服务

在 WebScout 仓库根目录下执行：

```bash
python -m protocol.a2a.app.main --host localhost --port 10000
```

| 命令行参数 | 默认值 | 说明 |
| --- | --- | --- |
| `--host` | `localhost` | 服务监听主机地址（同时写入 `AgentCard.url`）。 |
| `--port` | `10000` | 服务监听端口。 |

---

## 接口调用示例

### 1. 获取 Agent Card

```bash
curl http://localhost:10000/.well-known/agent.json
```

返回结果核心字段示例：

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

### 2. 发送任务（JSON-RPC `message/send`）

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
        "parts": [{"text": "使用 Python 计算前 10 个斐波那契数并打印结果"}]
      }
    }
  }'
```

### 3. 使用官方 A2A 示例客户端连接（可选）

也可以克隆 [`google-a2a/a2a-samples`](https://github.com/google-a2a/a2a-samples)，通过官方 CLI 或前端宿主注册 `http://localhost:10000`：

```bash
git clone https://github.com/google-a2a/a2a-samples.git
cd a2a-samples/samples/python/hosts/cli
uv run . --agent http://localhost:10000
```

---

## 参考链接

- [A2A 协议官方文档](https://google.github.io/A2A/#/documentation)
