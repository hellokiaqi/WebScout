# 图表可视化工具（Chart Visualization Tool）

[English](README.md) | 简体中文 | [日本語](README_ja.md) | [한국어](README_ko.md)

`app/tool/chart_visualization` 为 WebScout 的数据分析智能体 `DataAnalysis`（`app/agent/data_analysis.py`）提供数据处理与图表生成能力。工具链通过 Python（`pandas`）完成数据清洗与元数据准备，再通过子进程调用 TypeScript 脚本（`src/chartVisualize.ts`），借助 [@visactor/vmind](https://github.com/VisActor/VMind) 生成图表规格（Spec）与数据洞察，使用 [@visactor/vchart](https://github.com/VisActor/VChart) 渲染交互式 HTML 图表，或通过 `puppeteer` 导出静态 PNG 图片。

---

## 环境要求与安装

1. **Python 环境**：激活 WebScout 虚拟环境并安装根目录依赖（`uv pip install -r requirements.txt`，其中 `pandas` 会随 `datasets` 自动安装；若使用精简环境可手动执行 `uv pip install pandas`），并在 `config/config.toml` 中配置有效的 `[llm]` 接口。
2. **Node.js >= 18**：用于运行 `npx ts-node src/chartVisualize.ts` 及 `puppeteer`。

### macOS / Linux

```bash
# 安装并启用 Node.js >= 18（以 nvm 为例）
nvm install 22
nvm use 22

# 进入图表工具目录安装前端依赖
cd app/tool/chart_visualization
npm install
```

### Windows（PowerShell）

```powershell
# 安装并启用 Node.js >= 18（可通过 nvm-windows 或 Node.js 官网安装包）
nvm install 22
nvm use 22

# 进入图表工具目录安装前端依赖
cd app/tool/chart_visualization
npm install
```

---

## 工具说明

`DataAnalysis` 智能体组合了本目录下的三个核心工具（以及 `terminate`）：

### 1. `python_execute`（`python_execute.py` -> `NormalPythonExecute`）

执行通用数据分析与报表脚本（默认单次超时 `timeout = 5` 秒），用于数据集概览、指标计算、异常值筛查以及在 `workspace/` 下生成文本或 HTML 分析报告。

**输入参数**：

```json
{
  "code_type": "process | report | others",
  "code": "Python 代码字符串，需使用 print() 输出关键分析结果"
}
```

### 2. `visualization_preparation`（`chart_prepare.py` -> `VisualizationPrepare`）

在正式绑图前运行 Python 代码生成 CSV 数据与 JSON 配置文件，支持两种模式：

- **`code_type = "visualization"`（数据 -> 图表）**：将原始数据清洗并保存为 `.csv`，生成形如 `[{"csvFilePath": "path/to/data.csv", "chartTitle": "图表标题或描述"}]` 的 UTF-8 JSON 文件。
- **`code_type = "insight"`（图表 + 洞察 -> 图表标注）**：从已生成图表的洞察列表中挑选需要叠加标注的序号（从 1 开始），生成形如 `[{"chartPath": "path/to/chart.html", "insights_id": [1, 2]}]` 的 JSON 文件。

**输入参数**：

```json
{
  "code_type": "visualization | insight",
  "code": "用于生成 CSV 与 JSON 元数据并 print(json_path) 的 Python 代码"
}
```

### 3. `data_visualization`（`data_visualization.py` -> `DataVisualization`）

读取 `visualization_preparation` 生成的 JSON 配置文件，用 `pandas` 加载 CSV 数据后通过 `stdin`/`stdout` 调用 `npx ts-node src/chartVisualize.ts`。

**输入参数（`execute`）**：

```json
{
  "json_path": "workspace/chart_meta.json",
  "output_type": "html | png",
  "tool_type": "visualization | insight",
  "language": "en | zh"
}
```

- `output_type`：`DataVisualization.execute()` 默认值为 `"html"`（支持 VChart 交互渲染）；设为 `"png"` 时通过 Puppeteer 渲染静态图片（仅当底层 `src/chartVisualize.ts` 未收到该字段时才会回退为 `"png"`）。
- `tool_type`：`"visualization"` 用于生成图表并提取最多 6 条统计洞察（支持柱状图、折线图、面积图、散点图、双轴图）；`"insight"` 用于将选定的洞察标注写回已有图表。
- `language`：数据洞察输出语言，支持 `"en"`（默认）与 `"zh"`。

**本地输出产物**（统一保存在 `workspace/visualization/` 目录下）：

- `<file_name>.html` 或 `<file_name>.png`：最终图表文件。
- `<file_name>.json`：VChart 图表 Spec 与 VMind 洞察原始结构。
- `<file_name>.md`：按序号整理的 Markdown 数据洞察说明（当图表类型支持洞察提取时生成）。

---

## VMind 与渲染配置说明

- **大模型调用**：`DataVisualization.invoke_vmind()` 默认读取 `config.llm["default"]` 的 `base_url`、`model`、`api_key`，向 `${baseUrl}/chat/completions` 发起请求，因此 `[llm]` 需配置兼容 OpenAI Chat Completions 格式的接口。
- **尺寸与主题**：默认采用 VChart `"light"` 亮色主题（`theme: "light"`）。HTML 输出默认占满容器宽高（`100%`）；PNG 输出由无头浏览器渲染 Canvas 后导出。

---

## 使用方式与演示脚本

### 在 `run_flow.py` 中启用数据分析 Agent

修改 `config/config.toml` 中的 `[runflow]` 配置即可在多步规划流中挂载 `DataAnalysis`：

```toml
[runflow]
flow_type = "planning"
use_data_analysis_agent = true
```

随后运行 `python run_flow.py` 即可。

### 运行内置测试示例

`app/tool/chart_visualization/test/` 目录下提供了**两个**可直接运行的演示脚本（请在 WebScout 仓库根目录下执行）：

1. **多类型图表生成测试**（连续运行 9 组数据场景，涵盖分地区销量、品牌份额、销售趋势、热词热度、电动车散点对比、漏斗转化率、男女早餐对比、六维能力雷达图与流向图）：
   ```bash
   python -m app.tool.chart_visualization.test.chart_demo
   ```

2. **图文数据报告生成测试**（分析多团队月度工时数据并生成带图表的完整 HTML 分析报告）：
   ```bash
   python -m app.tool.chart_visualization.test.report_demo
   ```

运行完成后，生成的图表与报告均位于 `workspace/visualization/` 目录下。
