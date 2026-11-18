# Chart Visualization Tool

English | [简体中文](README_zh.md) | [日本語](README_ja.md) | [한국어](README_ko.md)

`app/tool/chart_visualization` powers WebScout's `DataAnalysis` agent (`app/agent/data_analysis.py`). It combines Python (`pandas`) data preparation with TypeScript chart generation (`src/chartVisualize.ts`), using [@visactor/vmind](https://github.com/VisActor/VMind) to generate chart specifications and statistical insights, [@visactor/vchart](https://github.com/VisActor/VChart) for interactive HTML rendering, and `puppeteer` for headless PNG export.

---

## Prerequisites & Installation

1. **Python dependencies**: Ensure the WebScout virtual environment is active (`uv pip install -r requirements.txt`; `pandas` is installed automatically via `datasets`, or run `uv pip install pandas` if using a minimal environment) and `config/config.toml` has valid `[llm]` credentials.
2. **Node.js >= 18**: Required to execute `npx ts-node src/chartVisualize.ts` and `puppeteer`.

### macOS / Linux

```bash
# Install Node.js >= 18 (using nvm or your system package manager)
nvm install 22
nvm use 22

# Install Node dependencies for chart visualization
cd app/tool/chart_visualization
npm install
```

### Windows (PowerShell)

```powershell
# Install Node.js >= 18 (e.g., via nvm-windows or nodejs.org)
nvm install 22
nvm use 22

# Install Node dependencies for chart visualization
cd app/tool/chart_visualization
npm install
```

---

## Included Tools

The `DataAnalysis` agent registers three tools from this package (plus `terminate`):

### 1. `python_execute` (`NormalPythonExecute` in `python_execute.py`)

Executes Python code (`timeout = 5` seconds by default) for data cleaning, statistical summaries, outlier checks, and Markdown/HTML report generation inside `workspace/`.

**Input Schema**:

```json
{
  "code_type": "process | report | others",
  "code": "Python source code using print() to surface results"
}
```

### 2. `visualization_preparation` (`VisualizationPrepare` in `chart_prepare.py`)

Runs Python code to prepare clean CSV datasets and JSON metadata consumed by `data_visualization`:

- **`code_type = "visualization"` (Data -> Chart)**: Cleans raw data into `.csv` files and writes a UTF-8 JSON file containing `[{"csvFilePath": "path/to/data.csv", "chartTitle": "Concise chart description"}]`.
- **`code_type = "insight"` (Chart + Insight -> Chart)**: Selects 1-based insight IDs from previously generated chart insights and writes `[{"chartPath": "path/to/chart.html", "insights_id": [1, 2]}]`.

**Input Schema**:

```json
{
  "code_type": "visualization | insight",
  "code": "Python script that writes the JSON metadata file and prints its path"
}
```

### 3. `data_visualization` (`DataVisualization` in `data_visualization.py`)

Reads the JSON metadata file from `visualization_preparation`, loads CSV records with `pandas`, and invokes `npx ts-node src/chartVisualize.ts` over `stdin`/`stdout`.

**Input Parameters (`execute`)**:

```json
{
  "json_path": "workspace/chart_meta.json",
  "output_type": "html | png",
  "tool_type": "visualization | insight",
  "language": "en | zh"
}
```

- `output_type`: Defaults to `"html"` in `DataVisualization.execute()` (interactive VChart HTML page). Set `"png"` to render a static PNG via Puppeteer (`src/chartVisualize.ts` falls back to `"png"` only when `output_type` is omitted from raw stdin JSON).
- `tool_type`: `"visualization"` generates charts and extracts up to 6 statistical insights (for bar, line, area, scatter, and dual-axis charts); `"insight"` updates an existing chart spec with selected insight annotations.
- `language`: Insight text language (`"en"` default, or `"zh"`).

**Generated Files** (saved under `workspace/visualization/`):

- `<file_name>.html` or `<file_name>.png`: Rendered chart.
- `<file_name>.json`: Serialized VChart specification and VMind insight objects.
- `<file_name>.md`: Numbered Markdown summary of discovered insights (when applicable).

---

## VMind & Rendering Settings

- **LLM Configuration**: `DataVisualization.invoke_vmind()` passes `config.llm["default"]` (`base_url`, `model`, `api_key`) to VMind (`${baseUrl}/chat/completions`). Use an OpenAI-compatible chat completions endpoint in `[llm]`.
- **Dimensions & Theme**: Uses VChart's `"light"` theme (`theme: "light"`). HTML charts expand to `100%` width and height unless explicit dimensions are passed; PNG exports capture the rendered canvas via Puppeteer.

---

## Usage & Demo Scripts

### Enable in `run_flow.py`

To include `DataAnalysis` alongside `Manus` in the planning flow, update `config/config.toml`:

```toml
[runflow]
flow_type = "planning"
use_data_analysis_agent = true
```

Then run `python run_flow.py`.

### Run Standalone Demo Scripts

Two runnable demo scripts are provided under `app/tool/chart_visualization/test/` (run from the WebScout repository root):

1. **Multi-Chart Generation Demo** (runs 9 chart scenarios covering bar, pie/share, line trend, word cloud/popularity, scatter, funnel, grouped comparison, radar, and flow data):
   ```bash
   python -m app.tool.chart_visualization.test.chart_demo
   ```

2. **Graphical HTML Data Report Demo** (analyzes multi-team monthly hours and generates a complete HTML report with embedded charts):
   ```bash
   python -m app.tool.chart_visualization.test.report_demo
   ```

All generated charts, specs, and reports are written to `workspace/visualization/`.
