# Commerce Report Delivery & Query Quality Optimization — Implementation Record

- **Date**: 2026-04-20
- **Status**: Landed in `app/commerce/executor.py`, `app/commerce/mcp_bridge.py`, `app/mcp/commerce_public_server.py`, `tests/commerce/test_executor_helpers.py`, and `tests/commerce/test_mcp_bridge.py`

---

## 1. Objective

Improve `product_compare_v2` execution latency and evidence precision in `public_only` mode by:

1. Preserving concrete product/model names and hardware configurations when deriving search queries.
2. Avoiding redundant external search-engine round-trips on policy-driven tasks when direct collectors and MCP tools have already run.
3. Eliminating duplicate Playwright browser snapshot attempts inside `CommerceMCPBridge` so browser recovery stays in a single owner (`CommerceResearchExecutor`).

---

## 2. Landed Implementation Details

### 2.1 Concrete Product Query Preservation (`app/commerce/executor.py` & `app/mcp/commerce_public_server.py`)

- `_derive_search_keyword()` and `build_search_queries()` in `app/commerce/executor.py` strip task boilerplate (`price`, `compare`, `official`, `reviews`, platform aliases) without stripping concrete product family/model tokens (such as `Google Pixel 9` or `iPhone 16`), and add a secondary quoted broad marketplace query (`"<Brand Model>" "<Platform>" unlocked new price -case ...`) when needed.
- `_clean_product_query(..., include_configuration=True)` in `app/mcp/commerce_public_server.py` retains explicit hardware configuration tokens (`size`, `chip`, `memory`, `storage`, `color`, `market`) via `_append_configuration_terms()` while excluding adjacent variants (`_product_variant_exclusions()`, e.g. excluding `"iPhone 16 Pro"`, `"Pixel 9 Pro XL"`, `"Galaxy S25 Ultra"` when the base model is requested).

### 2.2 Direct Platform Fallback Routing (`app/commerce/executor.py`)

- `should_prefer_direct_platform_fallback(task, execution_profile)`:
  - Under `product_compare_v2`, returns `True` for `source_role` in `{"official", "review_marketplace", "review_video", "review_community"}` as well as `marketplace` tasks with `strategy == "policy_direct"`.
  - Because Tier 1 (`_collect_product_compare_v2_observations`) already executes dedicated public search/mirror collection for `product_compare_v2` tasks, skipping a second generic `WebSearch` loop in Tier 3 avoids burning search timeouts on rate-limited engines.
- `should_browser_enrich_search_result(task, execution_profile, result)`:
  - Under `product_compare_v2`, skips slow headless browser enrichment on synthetic `platform_fallback` search URLs for `marketplace`, `review_marketplace`, and `review_community` roles, reserving browser navigation for `official` product pages or concrete search hits.

### 2.3 Single-Owner Browser Fallback (`app/commerce/mcp_bridge.py`)

- In `CommerceMCPBridge.collect(task)`, internal Playwright MCP snapshot helpers (`_collect_playwright_official_snapshot` and `_collect_playwright_marketplace_snapshot`) are skipped when `self.execution_profile == "product_compare_v2"`.
- Under `product_compare_v2`, `CommerceMCPBridge` focuses strictly on structured MCP data tools (such as `commerce_public` or `brightdata`), leaving browser-level fallback and diagnostics to `CommerceResearchExecutor`.

---

## 3. Regression Coverage & Verification

Key regression tests:

- `tests/commerce/test_executor_helpers.py` (query specificity, variant exclusions, `should_prefer_direct_platform_fallback`, and `should_browser_enrich_search_result` checks)
- `tests/commerce/test_mcp_bridge.py::test_product_compare_v2_mcp_skips_playwright_snapshot_fallback`

Run verification from the repository root:

```bash
python -m pytest tests/commerce/test_flow.py tests/commerce/test_executor_helpers.py tests/commerce/test_mcp_bridge.py -q
python -m py_compile app/commerce/executor.py app/commerce/mcp_bridge.py app/flow/commerce.py
```
