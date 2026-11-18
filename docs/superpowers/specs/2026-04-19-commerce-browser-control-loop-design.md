# Commerce Browser Control Loop — Technical Design Specification

- **Date**: 2026-04-19
- **Status**: Implemented (`app/flow/commerce.py`, `app/commerce/executor.py`, `app/commerce/models.py`, `app/commerce/mcp_bridge.py`, `app/commerce/grounding.py`)

---

## 1. Objective

Make WebScout's commerce research workflow (`CommerceDecisionFlow`) operate as an explicit, bounded `Planner -> Executor -> Reviewer` state machine with structured step contracts, predictable fallback ordering, failure-signature tracking, and transparent source-gap reporting:

- **Planner** normalizes free-form product requests, resolves brand/category source policy, and emits bounded `CommerceTask` steps carrying a `StepContract`.
- **Executor** executes one `CommerceTask` at a time in strict priority order: built-in direct public collectors -> optional MCP tools -> search/browser fallback.
- **Reviewer** evaluates step outcomes against each `StepContract`, records failure signatures, schedules at most one follow-up round while suppressing repeatedly failing platform roles, and synthesizes a `DecisionReport` that exposes blocked or missing sources instead of hiding them.

---

## 2. State & Component Architecture

### 2.1 LangGraph State (`CommerceGraphState` in `app/flow/commerce.py`)

`CommerceGraphState` separates persistent investigation state, step working state, and transient observations so context stays bounded across steps:

- **Persistent state**: `request_text`, `objective`, `constraints`, `global_plan`, `plan`, `evidence`, `verified_evidence`, `diagnostics`, `failure_summary` (capped at last 8 failure signatures).
- **Working step state**: `pending_tasks`, `completed_tasks`, `current_step`, `step_success_criteria`, `recent_action_trace` (capped at last 6 entries), `retry_count`, `step_evaluations`, `followup_rounds`.
- **Transient & environment state**: `transient_observation` (`last_platform`, `last_category`, `last_evidence_delta`, `last_diagnostic_reasons`), `environment` (`CommerceExecutionEnvironment`), `final_report`.

### 2.2 Planner (`_planner_node` & `_create_plan`)

Planner derives:

- **`ProductIdentity`** (`app/commerce/policy.py`): `brand`, `family`, `category`, `model_name`, `variant_tokens`, and `configuration` (`size`, `chip`, `memory`, `storage`, `color`, `market`).
- **Source roles**: `marketplace`, `official`, `review_marketplace`, `review_editorial`, `review_video`, and `review_community`.
- **Representative model baseline (`comparison_subject`)**: Narrows broad configuration-sensitive requests (such as `MacBook Pro` or `MacBook Air`) to one comparable baseline model when explicit configuration tokens are absent.
- **`StepContract`** (`app/commerce/models.py`): Attached automatically to every `CommerceTask` via `ensure_step_contract()` with:
  - `step_goal`
  - `allowed_action_types` (`direct_collect`, `mcp_collect`, `web_search`, `browser_navigate`, `browser_extract`, `visual_grounding`, `review_synthesis`, `verify`)
  - `target_object`
  - `success_criteria`
  - `fallback_policy`
  - `retry_limit`

Planner describes *what evidence* a step requires, never low-level element clicks.

### 2.3 Executor (`CommerceResearchExecutor` in `app/commerce/executor.py`)

`CommerceResearchExecutor.execute_task(task)` handles a single `CommerceTask` without re-planning the global graph:

1. **Tier 1 — Direct Public Collectors**: Calls profile-matched async collectors in `app/mcp/commerce_public_server.py`. Returns immediately when clean, priced, non-degraded observations satisfy the step.
2. **Tier 2 — Optional MCP Bridge (`_collect_mcp_evidence`)**: Calls `CommerceMCPBridge.collect(task)` when `should_collect_mcp()` allows MCP for the task's role and profile. Attaches `CommerceToolSchema` metadata (`name`, `input_schema`, `output_schema`, `timeout_seconds`, `error_codes`, `version`, `provider`) to resulting `EvidenceItem.metadata`.
3. **Tier 3 — Search & Browser Fallback**: Uses direct platform URLs (`build_platform_fallback_results`) or `WebSearch` (`build_search_queries`), optional `CommerceBrowserController` navigation/extraction, HTML fetch fallback (`WebContentFetcher`), session-browser retry (when `session_mode` is `auto` or `local_cdp`), and DOM/layout + vision grounding (`GuiPlusGrounder`).

Every rejected or blocked candidate appends a structured entry to `executor.last_task_diagnostics` (`platform`, `category`, `source_role`, `stage`, `reason`, `url`).

### 2.4 Step Contract Evaluation & Reviewer (`_evaluate_step_contract` & `_reviewer_node`)

Reviewer acts as a deterministic controller:

1. After each task, `_evaluate_step_contract()` checks `_task_evidence_matches()` and produces a `StepEvaluation`:
   - `status`: `"Success"`, `"Failed"`, or `"Unknown"`
   - `next_action`: `"complete"`, `"fallback"`, `"retry"`, or `"replan"`
   - `failure_signature`: `<category>:<platform>:<reason>` (for example `pricing:Amazon:login_required`)
2. `_update_failure_summary()` increments counts for repeated signatures.
3. In `_reviewer_node()`, if `followup_rounds < max_followup_rounds` (`1`), missing sources are scheduled for one follow-up pass, excluding already-attempted `(platform, source_role)` pairs and any pair suppressed by `_suppressed_platform_roles_from_failures()` (`FOLLOWUP_SUPPRESSION_FAILURE_COUNT = 2`).
4. Once follow-up tasks finish (or no eligible follow-up remains), `_build_report()` constructs `DecisionReport` and renders Markdown via `DecisionReport.to_markdown()`.

---

## 3. Diagnostic Classification & MCP Boundaries

### Standardized Failure Reasons

- Access & page state: `country_selector`, `login_required`, `verification_required`, `access_blocked`, `blank_page`, `home_shell_page`, `non_extractable_search_page`, `no_extractable_marketplace_results`
- Timeouts & runtime errors: `timeout`, `playwright_timeout`, `browser_enrichment_failed`, `exception`
- Evidence quality rejections: `model_mismatch`, `accessory_or_bundle`, `financing_or_monthly_price`, `trade_in_offer`, `no_evidence`

### MCP Policy

- MCP servers in `config/mcp.json` are optional enhancements; `stable_public_web` and `product_compare_v2` run without paid or login-gated MCP services.
- `MCPSettings.load_server_config()` skips disabled entries or servers missing `required_env_any` tokens without failing startup.
- Neither MCP tools nor browser modes attempt to bypass CAPTCHAs, login walls, or anti-bot protections.

---

## 4. Verification Suite

```bash
python -m pytest \
  tests/commerce/test_models.py \
  tests/commerce/test_flow.py \
  tests/commerce/test_executor_helpers.py \
  tests/commerce/test_mcp_bridge.py \
  tests/commerce/test_grounding.py -q
```
