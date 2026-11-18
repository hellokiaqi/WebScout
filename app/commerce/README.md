# `app/commerce` Module Reference

`app/commerce` implements the core data contracts, source-policy rules, browser session management, MCP tool bridge, DOM/vision grounding, and task execution pipeline for WebScout's product research and buying-decision agent.

For full CLI workflows, configuration recipes, and troubleshooting, see:

- [docs/commerce-agent.md](../../docs/commerce-agent.md) (English)
- [docs/commerce-agent.zh.md](../../docs/commerce-agent.zh.md) (简体中文)

---

## Module Map

| File | Key Classes & Functions | Responsibility |
| --- | --- | --- |
| `models.py` | `ProductIdentity`, `StepContract`, `StepEvaluation`, `CommerceTask`, `CommercePlan`, `PriceObservation`, `EvidenceItem`, `DecisionReport`, `CommerceExecutionEnvironment`, `GroundingCandidate`, `VisualGroundingHint`, `VisualPriceSignal`, `CommerceToolSchema` | Pydantic contracts shared across Planner, Executor, MCP Bridge, Grounder, and Reviewer, plus `DecisionReport.to_markdown()` rendering. |
| `policy.py` | `BrandSourcePolicy`, `POLICY_REGISTRY`, `detect_product_identity()`, `resolve_policy()`, `extract_product_configuration()`, `detect_requested_platforms()`, `split_requested_platforms()`, `is_configuration_sensitive_family()`, `build_configuration_signature()`, `has_configuration_conflict()` | Extracts brand, family, category, model, and hardware configuration (`size`, `chip`, `memory`, `storage`, `color`, `market`); maps brands/categories to official domains, marketplaces, and preferred MCP tools; detects cross-SKU conflicts. |
| `browser.py` | `CommerceBrowserController`, `BrowserMode` (`"public"` \| `"session"`) | Manages public and session browser runtimes (`local_browser_use`, `local_chrome_cdp`, `local_chrome_instance`, `daytona_sandbox`) under `public_only`, `auto`, and `local_cdp` modes, with `WebContentFetcher` HTML fallback when `extract_content` is unsupported. |
| `mcp_bridge.py` | `CommerceMCPBridge` | Connects to `stdio` and `sse` MCP servers from `config/mcp.json`, scores candidate tools per `CommerceTask`, generates `CommerceToolSchema` snapshots, enforces 20s per-tool timeouts, and normalizes outputs into structured observations. |
| `grounding.py` | `GuiPlusGrounder`, `build_grounding_candidates()` | Scores indexed DOM elements (`search_box`, `product_card`, `price_area`, `review_anchor`, `blocking_element`, `navigation`) with layout heuristic fallback, and optionally queries `config.llm["vision"]` to inspect page screenshots or recover local visible prices (`VisualPriceSignal`). |
| `prompts.py` | `PLANNER_SYSTEM_PROMPT`, `REVIEWER_SYSTEM_PROMPT`, `REVIEW_HIGHLIGHT_SYNTHESIZER_SYSTEM_PROMPT`, `GUI_PLUS_SYSTEM_PROMPT`, `VISUAL_PRICE_EXTRACTION_SYSTEM_PROMPT` | JSON-only system prompts for planning, review synthesis, and multimodal grounding. |
| `executor.py` | `CommerceResearchExecutor`, `build_search_queries()`, `build_platform_fallback_results()`, `compute_model_match_score()`, `detect_blocked_reason()`, `detect_unusable_price_reason()`, `parse_price()`, `score_source()` | Runs each `CommerceTask` through 3 tiers (direct public collectors -> MCP bridge -> search/browser fallback), filters accessories/renewed/mismatched models, and records task diagnostics in `last_task_diagnostics`. |

Companion modules outside `app/commerce/`:

- `app/flow/commerce.py`: `CommerceDecisionFlow` LangGraph state machine (`planner -> executor -> reviewer`), `StepContract` verification (`_evaluate_step_contract`), follow-up gating (`_suppressed_platform_roles_from_failures`), cross-SKU alignment (`_apply_series_scope_marketplace_alignment`), and report synthesis.
- `app/mcp/commerce_public_server.py`: `CommercePublicMCPServer` and built-in async public collectors (`_collect_marketplace_observations`, `_collect_official_observations`, `_collect_marketplace_review_observations`, `_collect_editorial_review_observations`, `_collect_youtube_review_observations`, `_collect_reddit_review_observations`).

---

## Core Data Contracts (`models.py`)

- **`ProductIdentity`**: `brand`, `family`, `category`, `model_name`, `variant_tokens`, `configuration`.
- **`StepContract`**: Attached automatically to every `CommerceTask` via `ensure_step_contract()`. Defines `step_goal`, `allowed_action_types`, `target_object`, `success_criteria`, `fallback_policy`, and `retry_limit`.
- **`CommerceTask`**: `category` (`pricing` \| `reviews` \| `official` \| `social`), `platform`, `query`, `goal`, `max_results`, `require_browser`, `source_role` (`marketplace`, `official`, `review_marketplace`, `review_video`, `review_community`, `review_editorial`), `strategy` (`policy_direct`, `mcp`, `search_browser`), `preferred_mcp_tools`, `requested_by_user`, `policy_id`, `step_contract`.
- **`CommercePlan`**: `product_name`, `normalized_query`, `comparison_subject`, `policy_id`, `product_identity`, `shopping_platforms`, `community_platforms`, `official_sources`, `requested_shopping_platforms`, `requested_review_platforms`, `unsupported_sources`, `decision_focus`, `tasks`.
- **`EvidenceItem`**: `category`, `platform`, `title`, `url`, `snippet`, `extracted_text`, `price` (`PriceObservation`), `sentiment`, `credibility`, `source_type`, `source_role`, `visual_hint` (`VisualGroundingHint`), `metadata` (tracks `model_match_score`, `offer_condition`, `quote_quality`, `browser_mode`, `mcp_tool`, `blocked_reason`, etc.).
- **`StepEvaluation`**: `platform`, `category`, `status` (`Success` \| `Failed` \| `Unknown`), `reason`, `evidence_count`, `success_criteria`, `next_action` (`complete` \| `retry` \| `fallback` \| `replan`), `failure_signature` (`category:platform:reason`).
- **`DecisionReport`**: Structured report rendered via `to_markdown()` with `status` (`complete` \| `partial` \| `incomplete`), `executive_summary`, `recommended_choice`, `confirmed_facts`, `rumors_or_uncertain`, `price_highlights`, `marketplace_quotes`, `official_baseline`, `review_highlights`, `evidence_matrix`, `sample_coverage`, `price_analysis`, `review_deep_dive`, `evidence_samples`, `unsupported_sources`, `source_notes`, `next_actions`, and `environment` (`CommerceExecutionEnvironment`).

---

## Task Execution Order (`executor.py`)

For every `CommerceTask`, `CommerceResearchExecutor.execute_task(task)` runs:

1. **Direct Public Collectors**: Invokes profile-matched collectors from `app/mcp/commerce_public_server.py` (`75s` timeout in `product_compare_v2`, `120s` for `review_editorial`, `35s` in `stable_public_web`). Returns early when clean, priced, non-degraded observations satisfy the step.
2. **Optional MCP Bridge (`_collect_mcp_evidence`)**: When `should_collect_mcp()` is true (`marketplace`, `review_community`, `official` in `product_compare_v2`; `pricing`, `reviews` in `stable_public_web`; all categories in `default`), calls `CommerceMCPBridge.collect(task)`. In `product_compare_v2`, `CommerceMCPBridge` skips internal Playwright snapshot helpers so browser fallback is handled once in stage 3.
3. **Search & Browser Fallback**:
   - Uses `build_platform_fallback_results(task)` directly when `should_prefer_direct_platform_fallback()` is true; otherwise queries `WebSearch` using `build_search_queries(task)`.
   - Enriches up to `MAX_BROWSER_ENRICHMENTS_PER_TASK = 1` candidate URL via `CommerceBrowserController` when `should_browser_enrich_search_result()` allows it.
   - If a `public` browser attempt hits `login_required`, `verification_required`, `access_blocked`, or `non_extractable_search_page` on a session-preferred site (`Amazon`, `Best Buy`, `Walmart`, `JD`, `Taobao`, `Tmall`, `Pinduoduo`, `Xiaohongshu`, `Weibo`) and a `session` browser is available (`auto` or `local_cdp`), retries once via `browser_mode="session"`.
   - Applies `GuiPlusGrounder.analyze()` or `extract_price_signal()` when `[llm.vision]` is configured and extracted text is under 160 characters or missing price text.

---

## Quick Verification

```bash
# Run all commerce unit and flow tests
python -m pytest tests/commerce -q

# Run a live public-only product comparison
python run_commerce.py \
  --execution-profile product_compare_v2 \
  --browser-session-mode public_only \
  --prompt "Compare iPhone 16 prices on Amazon and Best Buy, then summarize YouTube and Reddit feedback."
```
