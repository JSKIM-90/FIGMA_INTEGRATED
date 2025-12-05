# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This monorepo contains two interconnected projects for a visual web builder platform:

1. **Figma_Conversion/** - Figma-to-HTML/CSS conversion tool
2. **RNBT_architecture/** - Runtime framework for the visual web builder

---

## End-to-End Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Figma_Conversion (Figma MCP 필요)                                │
│                                                                      │
│     순수 역할: Figma 디자인 → 정적 HTML/CSS 추출                       │
│                                                                      │
│     작업:                                                            │
│     - Figma MCP로 디자인 정보 추출                                    │
│     - HTML/CSS 생성 (Tailwind → 순수 CSS)                            │
│     - 에셋 다운로드 (SVG, 이미지)                                      │
│     - Playwright 스크린샷 비교 (픽셀 검증)                             │
│                                                                      │
│     ❌ 스크립트 작업 없음 (순수성 유지)                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼  정적 HTML/CSS 전달
┌─────────────────────────────────────────────────────────────────────┐
│  2. RNBT_architecture (Figma MCP 불필요)                             │
│                                                                      │
│     역할: 정적 HTML/CSS → 동적 컴포넌트 변환 + 런타임 구성               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## RNBT_architecture 작업 프로세스

### Contract (약속) 정의

레이아웃 없이도 작업하기 위해 **Topic**과 **Event** 약속이 필요합니다.
이 약속을 정의하는 방식은 두 가지가 있습니다:

```
┌─────────────────────────────────────────────────────────────────────┐
│  약속 (Contract)                                                     │
│                                                                      │
│  1. Topic: 컴포넌트가 구독할 데이터                                    │
│     예: 'users', 'stats', 'alerts'                                   │
│                                                                      │
│  2. Event: 컴포넌트가 발행할 커스텀 이벤트                              │
│     예: '@userClicked', '@refreshRequested'                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 두 가지 접근법

#### 접근법 A: Top-Down (Page/기획 주도)

```
기획/Page 설계 → Contract 정의 → 컴포넌트가 Contract에 맞춰 구현
```

**장점**:
- 전체 데이터 흐름이 명확
- API 설계와 동시 진행 가능

**단점**:
- Page 설계가 완료되어야 컴포넌트 작업 시작 가능
- 컴포넌트 재사용성이 Page에 종속될 수 있음

#### 접근법 B: Bottom-Up (컴포넌트 주도)

```
Figma 컴포넌트 → 데이터 구조 추론 → 컴포넌트가 Contract 제안 → Page가 채택/조정
```

**장점**:
- Page 없이 컴포넌트 독립 개발 가능
- 컴포넌트 재사용성 극대화
- 함수형 프로그래밍 원칙: "함수가 기대하는 데이터 형태를 선언"

**단점**:
- 컴포넌트가 제안한 구조와 실제 API가 다를 수 있음 (어댑터 필요)
- Figma에서 추론 가능한 것은 샘플 수준 (필드명, 중첩 구조는 불확실)

**데이터 구조 추론 가능 범위**:
- ✅ 가능: 반복 패턴, 필드 개수, 대략적인 타입
- ⚠️ 불확실: 필드명, 중첩 구조, 실제 API 응답 형태

#### 권장: 상황에 따라 선택

| 상황 | 권장 접근법 |
|------|------------|
| 전체 페이지 설계가 확정됨 | A (Top-Down) |
| 컴포넌트 먼저 개발, 페이지는 나중 | B (Bottom-Up) |
| 재사용 가능한 컴포넌트 라이브러리 구축 | B (Bottom-Up) |
| API 스펙이 이미 존재 | A (Top-Down) |

### 독립 작업 가능한 영역

Contract가 정의되면 (어떤 접근법이든) **레이아웃 없이** 병렬 작업 가능:

```
        ┌──────────────────────┬──────────────────────┬───────────────────────┐
        ▼                      ▼                      ▼                       │
┌───────────────┐    ┌──────────────────┐    ┌───────────────────────┐       │
│  Component    │    │  Page Script     │    │  Data Layer           │       │
│               │    │                  │    │                       │       │
│  - HTML →     │    │  - before_load   │    │  - mock_server/       │       │
│    template   │    │  - loaded        │    │  - datasetList.json   │       │
│  - register.js│    │  - before_unload │    │                       │       │
│  - destroy.js │    │                  │    │                       │       │
│  - preview    │    │  - fetchAndPublish    │                       │       │
│               │    │  - eventHandlers │    │                       │       │
└───────────────┘    └──────────────────┘    └───────────────────────┘       │
        │                      │                      │                       │
        └──────────────────────┴──────────────────────┴───────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────┐
                    │  레이아웃 작업 (마지막)               │
                    │                                     │
                    │  - 전체 대시보드 구성                 │
                    │  - CONTAINER_STYLES.md 작성          │
                    │  - 컴포넌트 조립                      │
                    │  - Page가 Orchestrator 역할          │
                    └─────────────────────────────────────┘
```

### Bottom-Up 접근법 상세 (컴포넌트 주도)

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Figma 컴포넌트 분석                                               │
│                                                                      │
│     Figma 디자인에서 데이터 구조 추론:                                  │
│     - 반복되는 요소 → 배열                                            │
│     - 라벨+값 쌍 → 객체 필드                                          │
│     - 헤더 영역 → summary 객체                                        │
│                                                                      │
│     예: "구간별 성능현황" 컴포넌트                                      │
│     {                                                                │
│       header: { count: 681, total: 16959.0, avg: 0.603 },           │
│       items: [                                                       │
│         { label: '전체', tps: 61.0, responseTime: 0.001 },           │
│         ...                                                          │
│       ]                                                              │
│     }                                                                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. render 함수가 데이터 구조를 "선언"                                  │
│                                                                      │
│     function renderData(response) {                                  │
│         const { data } = response;                                   │
│         if (!data) return;                                           │
│                                                                      │
│         // 이 함수는 이런 구조를 기대한다고 명시                         │
│         const { header, items } = data;                              │
│         // ... 렌더링 로직                                            │
│     }                                                                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  3. mock 데이터로 preview.html 검증                                   │
│                                                                      │
│     컴포넌트가 독립적으로 동작하는지 확인                                │
│     Page 없이 mock 데이터만으로 렌더링 테스트                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. Page/API가 컴포넌트의 Contract에 맞춤                              │
│                                                                      │
│     - API가 컴포넌트 기대 구조로 응답                                   │
│     - 또는 Page에서 어댑터로 변환                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 컴포넌트 변환: 정적 → 동적

```html
<!-- Before: Figma에서 온 정적 HTML -->
<div class="user-card">
    <span class="name">홍길동</span>
    <span class="email">hong@example.com</span>
</div>

<!-- After: 동적 컴포넌트 (template 패턴) -->
<template id="user-card-template">
    <div class="user-card">
        <span class="name"></span>
        <span class="email"></span>
    </div>
</template>
<div class="user-list"></div>
```

### 메소드 시그니처 통일

```javascript
// fetchData 응답을 그대로 사용
function renderData({ response }) {
    const { data } = response;
    if (!data) return;  // Guard clause

    // 렌더링 로직
}
```

---

## Figma_Conversion

Converts Figma designs to HTML/CSS code using Figma MCP (Model Context Protocol).

### Commands

```bash
cd Figma_Conversion

# Install dependencies
npm install

# Run local server for preview
npm run serve

# Register Figma MCP server
claude mcp add figma-desktop --transport http --url http://127.0.0.1:3845/mcp

# Take screenshot with Playwright
node -e "const { chromium } = require('playwright'); ..."
```

### Key Workflow

1. User provides Figma link with `node-id`
2. Call `get_design_context` with `dirForAssetWrites` parameter (assets auto-download)
3. Call `get_screenshot` for visual comparison (original source of truth)
4. Convert Tailwind to pure CSS (use MCP values exactly, no guessing)
5. Capture implementation screenshot with Playwright
6. Compare and iterate until pixel-perfect match

### Critical Rules

- **No Guessing**: Use MCP-provided values exactly. Never estimate CSS values.
- **Asset Download Required**: Always use `dirForAssetWrites` parameter; never use localhost URLs directly in HTML
- **Pure Output**: Only HTML/CSS/assets. No scripts (responsibility of RNBT_architecture)

See `Figma_Conversion/CLAUDE.md` for detailed MCP rules and validation workflow.

---

## RNBT_architecture

Runtime framework for the visual web builder with functional programming patterns.

### Commands

```bash
cd RNBT_architecture/example_basic_01/mock_server
npm install
npm start  # Runs on port 3000

cd RNBT_architecture/example_master_01/mock_server
npm install
npm start  # Runs on port 3001
```

### Core Modules (Utils/)

| Module | Pattern | Purpose |
|--------|---------|---------|
| fx.js | Functional Programming | `go`, `map`, `filter`, `pipe`, `L.*` (lazy) |
| WEventBus.js | Pub-Sub | Component communication via `@eventName` |
| GlobalDataPublisher.js | Topic-based Pub-Sub | Data layer with `registerMapping`, `subscribe`, `fetchAndPublish` |
| WKit.js | Facade | Event binding, data fetch, 3D/2D utilities |

### Lifecycle

```
Page: before_load → [Component register] → loaded → [User Interaction] → before_unload
```

- **before_load**: Event handlers setup, Raycasting init
- **Component register**: Subscribe to topics, bind events
- **loaded**: Data publishing (`registerMapping` + `fetchAndPublish`)
- **before_unload**: Resource cleanup (1:1 with creation)

### Event Naming Convention

- Custom events: `@myClickEvent`, `@productSelected` (prefix with `@`)
- Native events: `click`, `submit` (no prefix)

### Example Architectures

- **example_basic_01**: Page-only architecture (IoT dashboard with polling)
- **example_master_01**: Master + Page architecture (dashboard with shared header/sidebar)
- **example_master_02**: Extended Master + Page pattern

See `RNBT_architecture/CLAUDE.md` for detailed patterns and best practices.

---

## Work Principles

### Before Making Changes

1. **Read existing files first** - Never write code without understanding current implementation
2. **Stay on purpose** - Don't add unrelated changes (e.g., styling when fixing logic)
3. **Wait for explicit instructions** - Observations are not commands; confirm before acting

---

## 웹 빌더 컴포넌트 구조

### 핵심 원칙

**Figma 링크 제공 = 컴포넌트 단위 선택**

사용자가 Figma 링크를 제공하면, 그것은 해당 요소를 컴포넌트로 선택한 것이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│  웹 빌더 기본 구조                                                    │
│                                                                      │
│  - 웹 빌더는 컴포넌트마다 div 컨테이너를 가짐 (기본 단위)                 │
│  - Figma 선택 요소의 가장 바깥 = div 컨테이너                          │
│  - Figma 선택 요소의 크기 = 컨테이너 크기                              │
│  - 내부 요소 = innerHTML (Figma 스타일 그대로)                        │
│                                                                      │
│  <div id="component-container">   ← Figma 선택 요소 크기              │
│      <!-- innerHTML -->           ← Figma 내부 요소 (스타일 그대로)    │
│  </div>                                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 컨테이너 크기 규칙

```
┌─────────────────────────────────────────────────────────────────────┐
│  컨테이너 크기 = Figma 선택 요소 크기 (고정)                           │
│                                                                      │
│  Figma에서 524 × 350 요소 선택                                        │
│       ↓                                                              │
│  #component-container {                                              │
│      width: 524px;                                                   │
│      height: 350px;                                                  │
│      overflow: auto;  /* 동적 렌더링 대응 */                          │
│  }                                                                   │
│                                                                      │
│  * 내부 HTML/CSS는 Figma 스타일 그대로 구현                            │
│  * overflow: auto는 동적 렌더링 시 콘텐츠 넘침 대응 (유일한 추가)        │
│  * 컨테이너에 추가 CSS 필요 시 → 컨테이너 선택자에 적용                  │
└─────────────────────────────────────────────────────────────────────┘
```

### CONTAINER_STYLES.md 역할

```
┌─────────────────────────────────────────────────────────────────────┐
│  CONTAINER_STYLES.md                                                 │
│                                                                      │
│  목적: 전체 레이아웃 조립 시 컨테이너 크기 재정의                        │
│                                                                      │
│  없음 → Figma 선택 요소 크기 그대로 사용 (고정)                         │
│  있음 → 레이아웃 기반 크기로 재정의                                     │
│         예: height: calc((100vh - 60px) / 2)                         │
│                                                                      │
│  * 둘 다 크기는 고정됨 (유동적이지 않음)                                │
│  * 차이: Figma 원본 기반 vs 레이아웃 기반                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Error Handling

- Use **Guard Clauses** for internal logic
- Use **try-catch only** for external library calls (e.g., ECharts, Tabulator)

```javascript
function renderData({ response }) {
    const { data } = response;
    if (!data) return;  // Guard clause

    try {
        this.chart.setOption(option);  // External library
    } catch (e) {
        console.error('[Chart]', e);
    }
}
```

---

## 실제 작업 예시: TimeTrendChart (시간대별 거래추이)

### 프로젝트 정보

- **Figma 프로젝트**: HANA_BANK_HIT_Dev
- **노드 ID**: 781:47816
- **컴포넌트 크기**: 831 × 350px
- **컴포넌트 타입**: page (ECharts Area Chart)

### 작업 흐름 (Bottom-Up 접근법)

```
1. Figma MCP로 디자인 정보 추출
   - get_design_context (dirForAssetWrites로 에셋 자동 다운로드)
   - get_screenshot (원본 비교용)

2. 데이터 구조 추론 (Bottom-Up)
   - 차트 시리즈 5개: 역대픽, 연중최고픽, 월픽, 전일, 금일
   - X축: 시간대 (00~22, 2시간 간격)
   - Y축: 수치 (0~1,800)
   - 필터 버튼: 전체, 상품, 1Q

3. Figma_Conversion: 정적 HTML/CSS 생성
   - Tailwind React → 순수 HTML/CSS 변환
   - Playwright 스크린샷 검증

4. RNBT_architecture: 동적 컴포넌트 변환
   - ECharts 기반 Area Chart 구현
   - Topic/Event 정의
   - preview.html로 독립 검증
```

### 생성된 파일 구조

```
Figma_Conversion/Conversion/HANA_BANK_HIT_Dev/
└── time-trend-chart/
    ├── assets/                    # 10개 SVG 에셋
    ├── screenshots/impl.png       # Playwright 검증
    ├── time-trend-chart.html      # 정적 HTML
    └── time-trend-chart.css       # 순수 CSS

RNBT_architecture/Projects/HANA_BANK_HIT_Dev/
└── page/components/TimeTrendChart/
    ├── views/component.html       # 동적 템플릿
    ├── styles/component.css       # 컴포넌트 CSS
    ├── scripts/
    │   ├── register.js            # ECharts 초기화 + 구독
    │   └── destroy.js             # 정리
    ├── preview.html               # 독립 검증
    └── preview_screenshot.png     # 검증 결과
```

### Contract 정의 (Bottom-Up 결과)

```javascript
// Topic
timeTrendData

// Event
@filterClicked

// 데이터 구조
{
  peakDate: "2025/08/05",
  series: {
    labels: ["00", "02", "04", "06", "08", "10", "12", "14", "16", "18", "20", "22"],
    역대픽: [1200, 1450, 950, ...],      // #526FE5
    연중최고픽: [1050, 1250, 880, ...],  // #52BEE5
    월픽: [550, 650, 500, ...],          // #009178
    전일: [280, 350, 280, ...],          // #52E5C3
    금일: [150, 200, 180, ...]           // #AAFD84
  },
  activeFilter: "전체"
}
```

### Projects 폴더 구조

실제 프로젝트 작업은 `example_xxx` 폴더가 아닌 `Projects` 폴더에서 진행:

```
RNBT_architecture/
├── example_basic_01/     # 예제 (참조용)
├── example_master_01/    # 예제 (참조용)
├── example_master_02/    # 예제 (참조용)
└── Projects/             # 실제 프로젝트
    └── HANA_BANK_HIT_Dev/
        └── page/
            └── components/
                └── TimeTrendChart/
```

---

## File Structure Summary

### Figma_Conversion Output

```
Figma_Conversion/
└── [ProjectName]/
    └── [ComponentName]/
        ├── assets/              # SVG, images
        ├── component.html       # Static HTML
        └── component.css        # Pure CSS
```

### RNBT_architecture Component

```
RNBT_architecture/
└── example_xxx/
    ├── CONTAINER_STYLES.md      # Container sizes (after layout)
    ├── datasetList.json         # API mappings
    ├── mock_server/             # Mock API
    └── page/
        ├── page_scripts/
        │   ├── before_load.js
        │   ├── loaded.js
        │   └── before_unload.js
        └── components/
            └── [ComponentName]/
                ├── views/component.html    # With template
                ├── styles/component.css
                ├── scripts/register.js
                ├── scripts/destroy.js
                └── preview.html
```
