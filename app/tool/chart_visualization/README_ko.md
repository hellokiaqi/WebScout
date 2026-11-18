# 차트 시각화 도구 (Chart Visualization Tool)

[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md) | 한국어

`app/tool/chart_visualization`은 WebScout의 데이터 분석 에이전트 `DataAnalysis`(`app/agent/data_analysis.py`)에서 사용하는 차트 생성 도구 모음입니다. Python(`pandas`)으로 데이터 전처리와 메타데이터 생성을 수행한 뒤, TypeScript 스크립트(`src/chartVisualize.ts`)에서 [@visactor/vmind](https://github.com/VisActor/VMind)를 호출해 차트 사양(Spec)과 데이터 인사이트를 생성하고, [@visactor/vchart](https://github.com/VisActor/VChart) 기반 대화형 HTML 또는 `puppeteer` 기반 PNG 이미지로 렌더링합니다.

---

## 사전 요구 사항 및 설치

1. **Python 환경**: WebScout 가상 환경을 활성화하고 루트 의존성을 설치한 뒤(`uv pip install -r requirements.txt`, 최소 환경인 경우 `uv pip install pandas` 추가 실행), `config/config.toml`의 `[llm]` 항목을 설정합니다.
2. **Node.js >= 18**: `npx ts-node src/chartVisualize.ts` 및 `puppeteer` 실행에 필요합니다.

### macOS / Linux

```bash
# Node.js >= 18 설치 및 활성화 (nvm 예시)
nvm install 22
nvm use 22

# 차트 시각화 패키지 의존성 설치
cd app/tool/chart_visualization
npm install
```

### Windows (PowerShell)

```powershell
# Node.js >= 18 설치 및 활성화 (nvm-windows 또는 공식 설치 프로그램)
nvm install 22
nvm use 22

# 차트 시각화 패키지 의존성 설치
cd app/tool/chart_visualization
npm install
```

---

## 제공 도구

`DataAnalysis` 에이전트는 다음 세 가지 도구(및 `terminate`)를 조합해 작업을 수행합니다.

### 1. `python_execute` (`python_execute.py` -> `NormalPythonExecute`)

데이터 정제, 기초 통계 계산, 이상치 탐지, 텍스트/HTML 분석 보고서 생성을 위한 Python 코드를 실행합니다(기본 타임아웃 `5`초).

**입력 스키마**:

```json
{
  "code_type": "process | report | others",
  "code": "print()로 분석 결과를 출력하는 Python 코드"
}
```

### 2. `visualization_preparation` (`chart_prepare.py` -> `VisualizationPrepare`)

Python 코드를 실행하여 `data_visualization` 도구가 읽을 CSV 파일과 JSON 설정 파일을 준비합니다.

- **`code_type = "visualization"` (데이터 -> 차트)**: 원본 데이터를 정제해 `.csv`로 저장하고 `[{"csvFilePath": "path/to/data.csv", "chartTitle": "차트 제목 또는 설명"}]` 형식의 UTF-8 JSON 파일을 생성합니다.
- **`code_type = "insight"` (차트 + 인사이트 -> 차트 주석)**: 생성된 차트 인사이트 번호(1부터 시작)를 선택해 `[{"chartPath": "path/to/chart.html", "insights_id": [1, 2]}]` 형식의 JSON 파일을 생성합니다.

**입력 스키마**:

```json
{
  "code_type": "visualization | insight",
  "code": "CSV 및 JSON 메타데이터 파일을 저장하고 경로를 print하는 Python 코드"
}
```

### 3. `data_visualization` (`data_visualization.py` -> `DataVisualization`)

`visualization_preparation`이 생성한 JSON 설정 파일을 읽고, `pandas`로 CSV 데이터를 변환한 뒤 `npx ts-node src/chartVisualize.ts`를 호출합니다.

**입력 파라미터 (`execute`)**:

```json
{
  "json_path": "workspace/chart_meta.json",
  "output_type": "html | png",
  "tool_type": "visualization | insight",
  "language": "en | zh"
}
```

- `output_type`: `DataVisualization.execute()`의 기본값은 `"html"`(대화형 VChart 페이지)이며, 정적 이미지가 필요할 때 `"png"`를 지정합니다(`src/chartVisualize.ts`에 `output_type`이 전달되지 않은 경우의 내부 기본값만 `"png"`입니다).
- `tool_type`: `"visualization"`은 차트 생성 및 최대 6개의 통계 인사이트 추출(막대, 선, 영역, 산점도, 이중축 차트 지원)을 수행하며, `"insight"`는 선택한 인사이트 주석을 기존 차트에 반영합니다.
- `language`: 인사이트 텍스트 언어(기본값 `"en"`, 또는 `"zh"`).

**출력 파일** (`workspace/visualization/` 디렉터리에 저장):

- `<file_name>.html` 또는 `<file_name>.png`: 렌더링된 차트 파일
- `<file_name>.json`: VChart Spec 및 VMind 인사이트 데이터
- `<file_name>.md`: 추출된 데이터 인사이트 Markdown 문서(해당 차트 유형인 경우)

---

## VMind 및 렌더링 설정

- **LLM 설정**: `DataVisualization.invoke_vmind()`는 `config.llm["default"]`(`base_url`, `model`, `api_key`)를 사용해 `${baseUrl}/chat/completions`를 호출합니다. OpenAI 호환 Chat Completions 엔드포인트를 설정하세요.
- **크기 및 테마**: 기본 테마는 VChart `"light"`(`theme: "light"`)입니다. HTML 출력은 기본적으로 전체 화면(`100%`)을 채우며, PNG 출력은 Puppeteer를 통해 캔버스를 이미지로 저장합니다.

---

## 사용 방법 및 데모 실행

### `run_flow.py`에서 활성화

`config/config.toml`의 `[runflow]`에서 `use_data_analysis_agent = true`로 설정하면 `PlanningFlow`에서 `DataAnalysis` 에이전트를 함께 사용할 수 있습니다.

```toml
[runflow]
flow_type = "planning"
use_data_analysis_agent = true
```

### 데모 스크립트 실행

`app/tool/chart_visualization/test/` 디렉터리에는 바로 실행할 수 있는 **2개**의 데모 스크립트가 포함되어 있습니다(WebScout 저장소 루트에서 실행하세요).

1. **다중 차트 생성 데모** (지역별 판매량, 브랜드 점유율, 매출 추이, 검색어 인기도, 전기차 산점도, 전환율 퍼널, 아침 식사 비교, 육각형 레이더, 데이터 흐름 등 9개 차트 시나리오 실행):
   ```bash
   python -m app.tool.chart_visualization.test.chart_demo
   ```

2. **HTML 그래픽 데이터 보고서 데모** (팀별 월간 근무 시간 데이터를 분석하여 차트가 포함된 HTML 보고서 생성):
   ```bash
   python -m app.tool.chart_visualization.test.report_demo
   ```

실행 결과물은 모두 `workspace/visualization/` 디렉터리에 저장됩니다.
