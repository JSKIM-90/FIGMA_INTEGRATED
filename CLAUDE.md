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

레이아웃 없이도 작업하기 위해 **Topic**과 **Event** 약속을 먼저 정의:

```
┌─────────────────────────────────────────────────────────────────────┐
│  약속 (Contract)                                                     │
│                                                                      │
│  1. Topic: 페이지가 publish할 데이터                                  │
│     예: 'users', 'stats', 'alerts'                                   │
│                                                                      │
│  2. Event: 페이지가 처리할 커스텀 이벤트                                │
│     예: '@userClicked', '@refreshRequested'                          │
│                                                                      │
│  * Claude가 기획자 역할로 Figma 디자인 보고 샘플 정의                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 독립 작업 가능한 영역

약속이 정의되면 **레이아웃 없이** 병렬 작업 가능:

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
                    │  레이아웃 작업 (나중에)               │
                    │                                     │
                    │  - 전체 대시보드 구성                 │
                    │  - CONTAINER_STYLES.md 작성          │
                    │  - 컴포넌트 조립                      │
                    └─────────────────────────────────────┘
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
