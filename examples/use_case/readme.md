# Use Case Examples

This directory contains sample outputs produced by the general-purpose `Manus` multi-tool agent (`main.py`), demonstrating web research, itinerary planning, and multi-format HTML artifact generation.

> For product price comparison and review research examples using the Commerce Agent (`run_commerce.py`), see [docs/commerce-agent.md](../../docs/commerce-agent.md).

---

## Japan 7-Day Travel Plan (`japan-travel-plan/`)

### Task Prompt

```text
I need a 7-day Japan itinerary for April 15-23 from Seattle, with a $2500-5000 budget for my fiancée and me. We love historical sites, hidden gems, and Japanese culture (kendo, tea ceremonies, Zen meditation). We want to see Nara's deer and explore cities on foot. I plan to propose during this trip and need a special location recommendation. Please provide a detailed itinerary and a simple HTML travel handbook with maps, attraction descriptions, essential Japanese phrases, and travel tips we can reference throughout our journey.
```

### Included Sample Artifacts

- `japan-travel-plan/japan_travel_handbook.html`: Full desktop/laptop HTML travel handbook with day-by-day itinerary, proposal planning guide, hotel recommendations, and budget breakdown.
- `japan-travel-plan/japan_travel_handbook_print.html`: Condensed, printer-friendly HTML version with quick-reference tables for paper travel folders.
- `japan-travel-plan/japan_travel_handbook_mobile.html`: Responsive mobile HTML handbook featuring collapsible sections, quick-access emergency contacts, and dark-mode support.
- `japan-travel-plan/japan_travel_guide_instructions.txt`: Version overview and usage notes generated alongside the handbooks.

### Screenshots

![Japan Travel Plan Preview 1](pictures/japan-travel-plan-1.png)

![Japan Travel Plan Preview 2](pictures/japan-travel-plan-2.png)

---

## Reproducing Locally

1. Configure your LLM API key in `config/config.toml` and install Playwright browsers (`playwright install chromium`).
2. From the WebScout repository root, run `main.py` with the prompt above:

```bash
python main.py --prompt "I need a 7-day Japan itinerary for April 15-23 from Seattle, with a $2500-5000 budget for my fiancée and me. We love historical sites, hidden gems, and Japanese culture (kendo, tea ceremonies, Zen meditation). We want to see Nara's deer and explore cities on foot. I plan to propose during this trip and need a special location recommendation. Please provide a detailed itinerary and a simple HTML travel handbook with maps, attraction descriptions, essential Japanese phrases, and travel tips we can reference throughout our journey."
```

3. Generated files are written to the `workspace/` directory in the repository root. You can open any `.html` file directly in a browser to inspect the result.
