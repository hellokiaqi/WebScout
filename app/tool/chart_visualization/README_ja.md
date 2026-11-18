# グラフ可視化ツール（Chart Visualization Tool）

[English](README.md) | [简体中文](README_zh.md) | 日本語 | [한국어](README_ko.md)

`app/tool/chart_visualization` は WebScout のデータ分析エージェント `DataAnalysis`（`app/agent/data_analysis.py`）向けのグラフ生成ツール群です。Python（`pandas`）でデータ整形とメタデータ作成を行い、TypeScript スクリプト（`src/chartVisualize.ts`）から [@visactor/vmind](https://github.com/VisActor/VMind) を呼び出してチャート仕様（Spec）とデータインサイトを生成し、[@visactor/vchart](https://github.com/VisActor/VChart) による対話型 HTML または `puppeteer` による PNG 画像として出力します。

---

## 前提条件とインストール

1. **Python 環境**：WebScout の仮想環境を有効化し、ルート依存関係をインストールしたうえで（`uv pip install -r requirements.txt`。最小環境の場合は `uv pip install pandas` も実行）、`config/config.toml` の `[llm]` を設定してください。
2. **Node.js >= 18**：`npx ts-node src/chartVisualize.ts` と `puppeteer` の実行に必要です。

### macOS / Linux

```bash
# Node.js >= 18 をインストール（nvm の例）
nvm install 22
nvm use 22

# チャート可視化ツールの依存パッケージをインストール
cd app/tool/chart_visualization
npm install
```

### Windows（PowerShell）

```powershell
# Node.js >= 18 をインストール（nvm-windows または公式インストーラ）
nvm install 22
nvm use 22

# チャート可視化ツールの依存パッケージをインストール
cd app/tool/chart_visualization
npm install
```

---

## ツール構成

`DataAnalysis` エージェントは以下の 3 つのツール（および `terminate`）を使用します。

### 1. `python_execute`（`python_execute.py` -> `NormalPythonExecute`）

データ前処理、基本統計量の算出、外れ値確認、テキスト／HTML レポート作成のための Python コードを実行します（デフォルトのタイムアウトは `5` 秒）。

**入力パラメータ**：

```json
{
  "code_type": "process | report | others",
  "code": "print() で分析結果を出力する Python コード"
}
```

### 2. `visualization_preparation`（`chart_prepare.py` -> `VisualizationPrepare`）

Python コードを実行し、`data_visualization` に渡す CSV ファイルと JSON 設定ファイルを生成します。

- **`code_type = "visualization"`（Data -> Chart）**：元データを整形して `.csv` に保存し、`[{"csvFilePath": "path/to/data.csv", "chartTitle": "グラフの説明"}]` 形式の UTF-8 JSON ファイルを出力します。
- **`code_type = "insight"`（Chart + Insight -> Chart）**：生成済みグラフのインサイト番号（1 始まり）を選択し、`[{"chartPath": "path/to/chart.html", "insights_id": [1, 2]}]` 形式の JSON ファイルを出力します。

**入力パラメータ**：

```json
{
  "code_type": "visualization | insight",
  "code": "CSV と JSON 設定ファイルを生成してパスを print する Python コード"
}
```

### 3. `data_visualization`（`data_visualization.py` -> `DataVisualization`）

`visualization_preparation` が作成した JSON ファイルを読み込み、`pandas` で CSV を変換して `npx ts-node src/chartVisualize.ts` を実行します。

**入力パラメータ（`execute`）**：

```json
{
  "json_path": "workspace/chart_meta.json",
  "output_type": "html | png",
  "tool_type": "visualization | insight",
  "language": "en | zh"
}
```

- `output_type`：`DataVisualization.execute()` のデフォルトは `"html"`（VChart 対話型ページ）です。静的画像を出力する場合は `"png"` を指定します（`src/chartVisualize.ts` 単体で `output_type` が省略された場合のフォールバックは `"png"`）。
- `tool_type`：`"visualization"` は新規グラフ生成と最大 6 件の統計インサイト抽出（棒・折れ線・面・散布図・二軸グラフ対象）を行い、`"insight"` は選択したインサイト注釈を既存グラフへ反映します。
- `language`：インサイトの出力言語（デフォルト `"en"`、または `"zh"`）。

**出力ファイル**（`workspace/visualization/` 配下に保存）：

- `<file_name>.html` または `<file_name>.png`：レンダリングされたグラフ
- `<file_name>.json`：VChart Spec と VMind インサイト情報
- `<file_name>.md`：抽出されたデータインサイトの Markdown 一覧（該当チャート種別のみ）

---

## VMind とレンダリング設定

- **LLM 設定**：`DataVisualization.invoke_vmind()` は `config.llm["default"]`（`base_url`、`model`、`api_key`）を使用して `${baseUrl}/chat/completions` を呼び出します。OpenAI 互換の Chat Completions エンドポイントを設定してください。
- **サイズとテーマ**：VChart の `"light"` テーマ（`theme: "light"`）を使用します。HTML 出力はデフォルトで画面全体（`100%`）に広がり、PNG 出力は Puppeteer で Canvas をキャプチャして保存します。

---

## 使い方とデモスクリプト

### `run_flow.py` での有効化

`config/config.toml` の `[runflow]` で `use_data_analysis_agent = true` を有効にすると、`PlanningFlow` から `DataAnalysis` エージェントを利用できます。

```toml
[runflow]
flow_type = "planning"
use_data_analysis_agent = true
```

### デモスクリプトの実行

`app/tool/chart_visualization/test/` には 2 つの動作確認用スクリプトが用意されています（WebScout リポジトリのルートから実行してください）。

1. **チャート生成デモ**（地域別売上、ブランドシェア、売上推移、検索ワード人気度、EV 散布図、コンバージョン率、朝食比較、レーダーチャート、データフローの計 9 タスクを実行）：
   ```bash
   python -m app.tool.chart_visualization.test.chart_demo
   ```

2. **HTML データレポート生成デモ**（チーム別月次稼働時間を分析し、グラフ付きの HTML レポートを生成）：
   ```bash
   python -m app.tool.chart_visualization.test.report_demo
   ```

生成されたすべてのファイルは `workspace/visualization/` に出力されます。
