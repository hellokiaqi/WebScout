# WebScout Developer & Maintenance Guide

This document defines the architecture boundaries, configuration contracts, runtime data flow, sandbox rules, and testing standards for contributors and coding agents working in WebScout.

---

## 1. Repository Architecture & Module Boundaries

WebScout is organized into two coexisting layers:

1. **Commerce Research & Decision Agent (Primary Maintained Path)**
   - `run_commerce.py`: CLI entrypoint (`--prompt`, `--execution-profile`, `--demo-profile`, `--browser-session-mode`, `--timeout-seconds`). Suppresses noisy library `stdout` during execution so `print(result)` emits clean Markdown to `stdout`, while logs go to `logs/commerce_<timestamp>.log`.
   - `app/flow/commerce.py`: `CommerceDecisionFlow` built on LangGraph (`StateGraph[CommerceGraphState]`) with `planner`, `executor`, and `reviewer` nodes.
   - `app/commerce/models.py`: Pydantic data contracts (`ProductIdentity`, `StepContract`, `StepEvaluation`, `CommerceTask`, `CommercePlan`, `PriceObservation`, `EvidenceItem`, `DecisionReport`, `CommerceExecutionEnvironment`, `GroundingCandidate`, `VisualGroundingHint`, `VisualPriceSignal`, `CommerceToolSchema`).
   - `app/commerce/policy.py`: Deterministic brand/family/category/configuration extraction (`detect_product_identity`, `extract_product_configuration`), brand source policies (`BrandSourcePolicy`, `POLICY_REGISTRY`, `resolve_policy`), and configuration-conflict checks (`has_configuration_conflict`).
   - `app/commerce/executor.py`: `CommerceResearchExecutor` implementing per-task collection, model/variant scoring, price hygiene filters, browser enrichment, and structured task diagnostics.
   - `app/mcp/commerce_public_server.py`: FastMCP server (`CommercePublicMCPServer`) and in-process public collector functions for marketplaces, official brand pages, retail customer reviews, editorial tech reviews, YouTube, and Reddit.
   - `app/commerce/mcp_bridge.py`: `CommerceMCPBridge` for optional external MCP tool discovery, ranking, input mapping, timeout enforcement, and observation normalization.
   - `app/commerce/browser.py`: `CommerceBrowserController` managing `public_only`, `auto`, and `local_cdp` browser runtimes.
   - `app/commerce/grounding.py` & `app/commerce/prompts.py`: `GuiPlusGrounder` DOM/layout candidate scoring plus multimodal vision assistance, alongside JSON-only system prompts.

2. **General OpenManus Agent & Infrastructure Layer**
   - `main.py` -> `app/agent/manus.py` (`Manus` general tool-calling agent).
   - `run_flow.py` -> `app/flow/flow_factory.py` (`PlanningFlow` or `CommerceDecisionFlow`).
   - `run_mcp.py` / `run_mcp_server.py` -> `app/agent/mcp.py` (`MCPAgent`) and `app/mcp/server.py` (`MCPServer`).
   - `sandbox_main.py` -> `app/agent/sandbox_agent.py` (`SandboxManus`) + `app/daytona/` + `app/tool/sandbox/`.
   - `app/sandbox/` -> Local Docker container sandbox (`LocalSandboxClient`, `SandboxManager`, `DockerSandbox`, `AsyncDockerizedTerminal`).
   - `app/tool/chart_visualization/` -> `DataAnalysis` Python + TypeScript (`@visactor/vmind`, `@visactor/vchart`, `puppeteer`) chart pipeline.
   - `protocol/a2a/` -> Experimental Google A2A JSON-RPC server (`A2AManus`, `ManusExecutor`).

Keep commerce-specific heuristics inside `app/commerce/`, `app/flow/commerce.py`, and `app/mcp/commerce_public_server.py`. Do not hardcode shopping or SKU rules into generic base classes (`BaseAgent`, `ToolCallAgent`, `BrowserUseTool`, `WebSearch`).

---

## 2. Configuration Loading (`app/config.py`)

`Config` is a thread-safe singleton initialized on import (`from app.config import config`):

- **Primary TOML file**: Loads `config/config.toml` if present; otherwise falls back to `config/config.example.toml`.
- **LLM profiles**: `[llm]` populates `config.llm["default"]`. Sub-tables such as `[llm.vision]` inherit missing fields from `[llm]` and register under `config.llm["vision"]`.
- **Browser settings (`[browser]`)**: Parsed into `BrowserSettings` (`headless`, `disable_security`, `session_mode`, `extra_chromium_args`, `chrome_instance_path`, `wss_url`, `cdp_url`, `proxy`, `max_content_length`). `config.set_browser_session_mode()` overrides `session_mode` at runtime when `--browser-session-mode` is passed.
- **MCP servers (`config/mcp.json`)**: `MCPSettings.load_server_config()` reads `config/mcp.json` when it exists. Entries with `"disabled": true`, unsupported transport types (anything other than `"stdio"` or `"sse"`), or unsatisfied `required_env_any` environment variables are skipped without failing startup.
- **Sandboxes**:
  - `[sandbox]` configures local Docker containers (`SandboxSettings`, defaults to `use_sandbox = False`).
  - `[daytona]` configures remote Daytona sandboxes (`DaytonaSettings`, `None` unless `[daytona]` is present).
- **Secret hygiene**: Never place real API keys or tokens in `config/*.example.*` or tracked files. `config/config.toml`, `config/mcp.json`, `logs/`, and `workspace/` must stay git-ignored.

---

## 3. Commerce Data Flow & Verification Pipeline

`CommerceDecisionFlow.execute(input_text)` processes requests through concrete state transitions in `CommerceGraphState`:

1. **Plan Construction (`_planner_node` -> `_create_plan`)**:
   - `stable_public_web`: Deterministic 4-task plan (Amazon, Best Buy, YouTube, Apple.com).
   - `product_compare_v2`: Calls `detect_product_identity()` and `resolve_policy()`. For broad configuration-sensitive families without explicit specs (currently `MacBook Pro` and `MacBook Air`), sets `CommercePlan.comparison_subject` to a single comparable baseline model. Emits `CommerceTask` items across `marketplace`, `official`, `review_marketplace`, `review_editorial`, `review_video`, and `review_community` roles, each populated with a validated `StepContract`.
   - `default`: Uses `_heuristic_plan_from_request()` when product and platforms are explicit; otherwise calls the Planner LLM (`PLANNER_SYSTEM_PROMPT`) with fallback to `_fallback_plan()`.

2. **Single-Task Execution (`_executor_node` -> `CommerceResearchExecutor.execute_task`)**:
   - **Step A — Direct Collectors**: Calls profile-specific async collectors from `app/mcp/commerce_public_server.py`. Returns early if clean, priced, non-degraded observations satisfy the task.
   - **Step B — MCP Bridge**: Calls `_collect_mcp_evidence()` when `should_collect_mcp()` allows it. Normalizes tool outputs into `EvidenceItem` objects with `CommerceToolSchema` metadata. In `product_compare_v2`, `CommerceMCPBridge` skips internal Playwright snapshot helpers so browser fallback happens only once in Step C.
   - **Step C — Search & Browser Fallback**: Uses direct platform URLs (`build_platform_fallback_results`) when `should_prefer_direct_platform_fallback()` is true, or runs `WebSearch` queries (`build_search_queries`). When browser enrichment is enabled for a candidate URL, navigates via `CommerceBrowserController`, extracts page text (falling back to `WebContentFetcher`), checks for access blocks (`login_required`, `verification_required`, `access_blocked`, `country_selector`, `blank_page`), optionally retries once via the `session` browser in `auto`/`local_cdp` mode, and runs `GuiPlusGrounder` when `[llm.vision]` is configured and text extraction is sparse or missing a price.
   - **Step D — Quality & Alignment Gates**: Filters unusable prices (`detect_unusable_price_reason`) and low model-match scores (`compute_model_match_score` vs `minimum_model_match_score`), appending every rejection or block reason to `executor.last_task_diagnostics`.

3. **Step Verification & Follow-Up Control (`_evaluate_step_contract` & `_reviewer_node`)**:
   - After every task, `_evaluate_step_contract()` emits a `StepEvaluation` (`Success` / `Failed` / `Unknown`, `next_action`, and `failure_signature` formatted as `category:platform:reason`).
   - `_executor_node` updates `verified_evidence`, `recent_action_trace` (bounded to last 6 entries), and `failure_summary` (bounded to last 8 signatures).
   - Once `pending_tasks` is empty, `_reviewer_node` checks coverage. If `followup_rounds < max_followup_rounds` (default `1`), it derives missing follow-up tasks while excluding already-attempted `(platform, source_role)` pairs and any pair suppressed by `_suppressed_platform_roles_from_failures()` (cumulative failure count >= `2`).

4. **Report Synthesis (`_build_report` -> `DecisionReport.to_markdown`)**:
   - Separates clean marketplace quotes from `degraded_marketplace` offers (renewed/refurbished, cross-region, or cross-SKU series mismatches via `_apply_series_scope_marketplace_alignment`).
   - Computes `DecisionReport.status` (`complete`, `partial`, or `incomplete`) from satisfied source requirements and `unsupported_sources`.
   - Renders deterministic Chinese/English report sections including price observations, official baseline, evidence matrix, review highlights, sample coverage, price analysis, review deep dive, sample excerpts, source notes, next actions, and runtime environment metadata.

---

## 4. Access Control & Anti-Bot Boundaries

- Never add code that attempts to solve or bypass CAPTCHAs, human-verification gates, login walls, paywalls, or anti-bot challenges.
- `GuiPlusGrounder` (`app/commerce/grounding.py`) is strictly for identifying product cards, price regions, review anchors, and detecting blocking overlays so the executor can classify the block reason accurately.
- When a source blocks automated access, record the diagnostic reason (`login_required`, `verification_required`, `access_blocked`, `country_selector`, `timeout`) and expose the gap in `DecisionReport`.

---

## 5. Sandbox Subsystems

- **Local Docker Sandbox (`app/sandbox/`)**:
  - `DockerSandbox` runs `python:3.12-slim` (configurable via `[sandbox]`) with memory/CPU quotas, optional bridge/none networking, path-traversal checks (`_safe_resolve_path`), tar-stream file copy, and `AsyncDockerizedTerminal`.
  - Managed via `SANDBOX_CLIENT` (`LocalSandboxClient`) or `SandboxManager` (concurrent locks + idle container cleanup).
- **Daytona Cloud Sandbox (`app/daytona/` & `app/tool/sandbox/`)**:
  - Configured via `[daytona]` in `config/config.toml`. Note that `app/daytona/sandbox.py` and `app/daytona/tool_base.py` read `config.daytona` at module import time, so only import `app.daytona.*` lazily when `config.daytona` is configured (see `app/commerce/browser.py` for the lazy-import pattern).

---

## 6. Testing & Pre-Merge Verification

`pytest.ini` sets `asyncio_default_fixture_loop_scope = function` and `tests/conftest.py` adds the repository root to `sys.path`.

- **Commerce unit & flow suite (no network or API keys required)**:
  ```bash
  python -m pytest tests/commerce -q
  ```
- **Targeted test files during iteration**:
  - `tests/commerce/test_models.py`: Pydantic contracts, default `StepContract`, Markdown rendering.
  - `tests/commerce/test_browser.py`: `public_only`, `auto`, `local_cdp` session resolution.
  - `tests/commerce/test_grounding.py`: DOM and layout fallback candidate scoring.
  - `tests/commerce/test_mcp_bridge.py`: `mcp.json` env filtering, tool ranking, schema snapshots, Playwright snapshot parsing.
  - `tests/commerce/test_executor_helpers.py`: Search query building, model/variant matching, price filtering, 3-tier execution order, public collector parsers.
  - `tests/commerce/test_flow.py`: Planner profiles, follow-up suppression, series-scope alignment, report status and Markdown sections.
- **Docker sandbox suite (requires local Docker daemon)**:
  ```bash
  python -m pytest tests/sandbox -q
  ```
- **Before finishing any change**:
  1. Run `python -m pytest tests/commerce -q`.
  2. Run `python -m py_compile` on touched Python modules.
  3. Check `git status` to verify no credentials, logs, or temporary workspace artifacts are staged.
