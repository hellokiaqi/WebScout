# Commerce Browser Control Loop — Implementation Record

- **Date**: 2026-04-19
- **Status**: Landed in `app/commerce/executor.py`, `app/flow/commerce.py`, `tests/commerce/test_executor_helpers.py`, and `tests/commerce/test_flow.py`
- **Related Spec**: [2026-04-19-commerce-browser-control-loop-design.md](../specs/2026-04-19-commerce-browser-control-loop-design.md)

---

## 1. Summary of Changes

This change aligned the runtime behavior of `CommerceResearchExecutor` and `CommerceDecisionFlow` with the 3-tier collection order and failure-aware follow-up gating defined in the control-loop specification:

1. **MCP Before Search/Browser Fallback (`app/commerce/executor.py`)**: Extracted `_collect_mcp_evidence()` and moved MCP invocation ahead of external web search and browser fallback so structured MCP tools run immediately when direct public collectors return empty or degraded marketplace quotes.
2. **Repeated-Failure Follow-Up Suppression (`app/flow/commerce.py`)**: Added failure-signature aggregation so `_reviewer_node()` suppresses follow-up tasks for `(platform, source_role)` pairs that have already reached the failure threshold.

---

## 2. Landed Implementation Details

### 2.1 Executor Tier Ordering (`app/commerce/executor.py`)

- **`CommerceResearchExecutor._collect_mcp_evidence(task, diagnostics)`**:
  - Checks `should_collect_mcp(task, self.execution_profile)` (`marketplace`, `review_community`, and `official` in `product_compare_v2`; `pricing` and `reviews` in `stable_public_web`; all roles in `default`).
  - Enforces profile-aware timeouts via `get_mcp_collection_timeout_seconds(task, self.execution_profile)` (`35s` for `product_compare_v2` marketplace tasks, `45s` for other `product_compare_v2` tasks, `35s` default).
  - Copies `self.mcp_bridge.last_diagnostics` into `diagnostics` under `stage="mcp_collection"`.
  - Normalizes observations into `EvidenceItem` instances, parses or validates `PriceObservation`, filters unusable prices via `detect_unusable_price_reason()`, scores model alignment via `compute_model_match_score()` against `minimum_model_match_score()`, and infers review sentiment for `reviews` / `social` tasks.
- **`CommerceResearchExecutor.execute_task(task)` sequence**:
  1. Run profile direct collectors (`_collect_product_compare_v2_observations` or `_collect_stable_public_observations`). Return early unless an `official` task lacks a priced baseline or `should_continue_collecting_after_marketplace_evidence()` indicates only degraded marketplace quotes were found.
  2. Run `_collect_mcp_evidence(task, diagnostics)`. Return early if MCP yields usable evidence and `should_continue_collecting_after_marketplace_evidence()` is `False`.
  3. Fall back to direct platform URLs (`build_platform_fallback_results` when `should_prefer_direct_platform_fallback()` is true) or multi-query `WebSearch` (`build_search_queries`) + browser enrichment (`_enrich_item_with_browser`), terminating cleanly once `should_continue_collecting_marketplace_evidence()` is satisfied.

### 2.2 Failure-Signature Follow-Up Gating (`app/flow/commerce.py`)

- **Constants**:
  - `FOLLOWUP_SUPPRESSION_FAILURE_COUNT = 2`
  - `SOURCE_ROLE_BY_CATEGORY = {"pricing": "marketplace", "official": "official", "reviews": "review_video", "social": "review_community"}`
- **`_suppressed_platform_roles_from_failures(failure_summary)`**:
  - Parses signatures formatted as `<category>:<platform>:<reason>`.
  - Sums `count` across entries for each `(platform, source_role)` pair and returns pairs whose cumulative failure count is `>= FOLLOWUP_SUPPRESSION_FAILURE_COUNT`.
- **Reviewer integration**:
  - `_reviewer_node()` passes `completed_tasks` and `failure_summary=state.get("failure_summary", [])` into `_derive_stable_public_web_followup_tasks()`, `_derive_product_compare_v2_followup_tasks()`, and `_derive_followup_tasks()`.
  - `_derive_product_compare_v2_followup_tasks()` merges `_suppressed_platform_roles_from_failures(failure_summary)` into `attempted_platform_roles`, preventing wasted follow-up retries against blocked platforms while still allowing untried platforms to run.

---

## 3. Regression Coverage & Verification

Key regression tests covering this behavior:

- `tests/commerce/test_executor_helpers.py::test_product_compare_v2_executor_tries_mcp_before_search_browser_fallback`
- `tests/commerce/test_flow.py::test_product_compare_v2_followup_suppresses_repeated_failed_platform_role`
- `tests/commerce/test_flow.py::test_product_compare_v2_followup_does_not_retry_attempted_marketplace`

Run verification from the repository root:

```bash
python -m pytest tests/commerce/test_flow.py tests/commerce/test_executor_helpers.py tests/commerce/test_mcp_bridge.py -q
python -m py_compile app/commerce/executor.py app/flow/commerce.py
```
