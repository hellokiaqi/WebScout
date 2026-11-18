# Commerce Report Depth Optimization — Implementation Record

- **Date**: 2026-04-20
- **Status**: Landed in `app/commerce/models.py`, `app/flow/commerce.py`, and `tests/commerce/test_flow.py`

---

## 1. Objective

Upgrade `product_compare_v2` outputs from a brief bullet summary into an evidence-backed buying-decision report by increasing public review sampling depth and adding deterministic analytical sections to `DecisionReport`.

---

## 2. Landed Implementation Details

### 2.1 Expanded Review Sampling Targets (`app/flow/commerce.py`)

`CommerceDecisionFlow._build_product_compare_v2_plan()` uses dedicated per-channel sample budgets so review synthesis has enough material even when individual items are filtered out for model/variant mismatch:

- `PRODUCT_COMPARE_V2_VIDEO_SAMPLE_COUNT = 30` (`source_role = "review_video"`, e.g. YouTube)
- `PRODUCT_COMPARE_V2_COMMUNITY_SAMPLE_COUNT = 24` (`source_role = "review_community"`, e.g. Reddit)
- `PRODUCT_COMPARE_V2_MARKETPLACE_REVIEW_SAMPLE_COUNT = 12` (`source_role = "review_marketplace"`, retail customer ratings/reviews)
- `PRODUCT_COMPARE_V2_EDITORIAL_REVIEW_SAMPLE_COUNT = 4` (`source_role = "review_editorial"`, open-web editorial reviews)
- `PRODUCT_COMPARE_V2_REVIEW_LLM_SAMPLE_LIMIT = 36` (maximum combined review excerpts passed to `REVIEW_HIGHLIGHT_SYNTHESIZER_SYSTEM_PROMPT`)

### 2.2 Structured Report Fields & Markdown Sections (`app/commerce/models.py`)

`DecisionReport` includes structured list fields rendered deterministically by `DecisionReport.to_markdown()`:

- `evidence_matrix`: Rendered under `## 证据矩阵` (Markdown table or bullets summarizing planned vs. collected status across marketplaces, official baselines, retail reviews, editorial reviews, video reviews, and community threads).
- `sample_coverage`: Rendered under `## 样本覆盖` (counts and platform breakdown per evidence role).
- `price_analysis`: Rendered under `## 价格与配置分析` (lowest/highest clean marketplace quotes, spread vs. official baseline, and caveats on degraded/renewed/unaligned quotes).
- `review_deep_dive`: Rendered under `## 口碑深挖` (cross-channel sentiment distribution and divergence between retail buyers, editorial reviewers, video creators, and community owners).
- `evidence_samples`: Rendered under `## 样本摘录` (compact attributed excerpts with source URLs across price, official, retail, editorial, video, and community items).

### 2.3 Deterministic Report Builders (`app/flow/commerce.py`)

In `_build_product_compare_v2_report()`, deterministic helper functions populate all analytical sections directly from verified `EvidenceItem` collections so report completeness does not degrade when LLM synthesis falls back:

- `_build_product_compare_v2_evidence_matrix()`
- `_build_product_compare_v2_sample_coverage()`
- `_build_product_compare_v2_price_analysis()`
- `_build_product_compare_v2_review_deep_dive()`
- `_build_product_compare_v2_evidence_samples()`
- `_build_product_compare_v2_review_highlights()` (synthesizes 6 structured Chinese takeaways—core strengths, complaints/controversies, price/upgrade value, long-term risks, target user profile, and one-line verdict—falling back to `_build_review_highlights()` if the LLM call fails or times out).

---

## 3. Regression Coverage & Verification

Covered in `tests/commerce/test_flow.py` (including deep review sample count assertions and rich Markdown section assertions on `DecisionReport.to_markdown()`).

```bash
python -m pytest tests/commerce/test_flow.py tests/commerce/test_models.py -q
python -m py_compile app/commerce/models.py app/flow/commerce.py
```
