# Daytona Cloud Sandbox Integration

`app/daytona` integrates [Daytona](https://www.daytona.io/) remote cloud sandboxes into WebScout so agents can execute shell commands, edit files, inspect images, and drive a full Chromium desktop browser with live VNC preview inside an isolated remote container.

---

## Architecture & Key Files

- `sandbox_main.py`: CLI entrypoint (`--prompt`) that creates and runs `SandboxManus` and deletes the remote sandbox on exit.
- `app/agent/sandbox_agent.py`: `SandboxManus` agent class. Initializes configured MCP servers plus a Daytona sandbox (`create_sandbox`), logs the VNC (`6080`) and web server (`8080`) preview URLs, and registers four sandbox tools:
  - `sandbox_browser` (`app/tool/sandbox/sb_browser_tool.py` -> `SandboxBrowserTool`)
  - `sandbox_files` (`app/tool/sandbox/sb_files_tool.py` -> `SandboxFilesTool`)
  - `sandbox_shell` (`app/tool/sandbox/sb_shell_tool.py` -> `SandboxShellTool`)
  - `sandbox_vision` (`app/tool/sandbox/sb_vision_tool.py` -> `SandboxVisionTool`)
- `app/daytona/sandbox.py`: Manages Daytona SDK client initialization (`Daytona`, `DaytonaConfig`), container creation (`create_sandbox` with 2 vCPU, 4 GB RAM, 5 GB disk, `1024x768x24` display, Chrome remote debugging on port `9222`, `auto_stop_interval=15`), `supervisord` session startup (`start_supervisord_session`), restart (`get_or_start_sandbox`), and deletion (`delete_sandbox`).
- `app/daytona/tool_base.py`: `SandboxToolsBase` base class for `/workspace`-scoped path sanitization (`clean_path`) and lazy sandbox startup/recovery.
- **Commerce Agent Integration (`app/commerce/browser.py`)**: When `[daytona]` is configured in `config/config.toml`, `CommerceBrowserController` automatically selects `SandboxBrowserTool` (`backend_name = "daytona_sandbox"`) as its public browser backend, falling back to local `BrowserUseTool` if initialization fails.

---

## Prerequisites

1. Python 3.12 virtual environment with repository dependencies installed (`daytona~=0.21.8` and `structlog~=25.4.0` are included in the root `requirements.txt`):
   ```bash
   uv pip install -r requirements.txt
   ```
2. A Daytona account and API key from [https://app.daytona.io/dashboard/keys](https://app.daytona.io/dashboard/keys).

---

## Setup & Configuration

1. Copy the Daytona configuration template from the WebScout repository root:
   ```bash
   cp config/config.example-daytona.toml config/config.toml
   ```

2. Edit `config/config.toml` to fill in your `[llm]`, `[llm.vision]`, and `[daytona]` settings:
   ```toml
   [daytona]
   daytona_api_key = "YOUR_DAYTONA_API_KEY"
   daytona_server_url = "https://app.daytona.io/api"
   daytona_target = "us"                           # "us" or "eu"
   sandbox_image_name = "whitezxj/sandbox:0.1.0"   # Prebuilt image with Chromium, VNC, and supervisord
   sandbox_entrypoint = "/usr/bin/supervisord -n -c /etc/supervisor/conf.d/supervisord.conf"
   VNC_password = "YOUR_VNC_PASSWORD"              # Defaults to "123456" in DaytonaSettings if omitted
   ```

> Keep `sandbox_image_name` and `sandbox_entrypoint` aligned with an image that runs `supervisord`, VNC (`6080`), and the sandbox browser service; otherwise `sandbox_browser` and preview links will not respond.

---

## Running `SandboxManus`

Run interactively (the CLI will prompt `Enter your prompt: `):

```bash
python sandbox_main.py
```

Or pass `--prompt` directly:

```bash
python sandbox_main.py --prompt "Search Nanjing travel information on https://hk.trip.com/travel-guide/guidebook/nanjing-9696/ and generate a travel guide saved to /workspace/index.html"
```

### Inspecting Live Execution & Outputs

When the sandbox starts (`start_supervisord_session` waits ~25 seconds for container services to initialize), WebScout prints two preview links in the terminal:

- **VNC URL (port `6080`)**: Watch live browser actions performed by `sandbox_browser` (unlock with `VNC_password`).
- **Website URL (port `8080`)**: Open HTML/web files served from `/workspace` inside the sandbox.
- **Daytona Dashboard**: Inspect running sandboxes and tmux shell sessions at [https://app.daytona.io/dashboard/sandboxes](https://app.daytona.io/dashboard/sandboxes).

When `sandbox_main.py` finishes, `agent.cleanup()` disconnects MCP sessions and deletes the Daytona sandbox automatically.

---

## References

- [Daytona Documentation](https://www.daytona.io/docs/)
