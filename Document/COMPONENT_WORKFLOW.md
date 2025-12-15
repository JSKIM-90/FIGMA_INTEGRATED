# Figma Design to Dynamic Component Workflow

Figma 디자인을 독립적인 동적 컴포넌트로 변환하는 End-to-End 워크플로우 가이드

---

## 1. 개요

### 1.1 목적

이 문서는 Figma에서 디자인된 UI 요소를 **독립적으로 동작하는 동적 컴포넌트**로 변환하는 전체 과정을 설명합니다.

### 1.2 핵심 가치

| 가치 | 설명 |
|------|------|
| **컴포넌트 독립성** | 페이지 컨텍스트 없이 컴포넌트가 독립적으로 개발/테스트 가능 |
| **관심사 분리** | 페이지는 조율(Orchestration), 컴포넌트는 렌더링에 집중 |
| **재사용성** | 동일 컴포넌트를 다양한 데이터 소스와 페이지에서 활용 |

### 1.3 워크플로우 전체 흐름

```
┌─────────────┐    ┌───────────┐    ┌──────────────┐    ┌─────────────────┐    ┌─────────┐
│ Figma Design │ → │ Figma MCP │ → │ Static HTML/ │ → │ Dynamic Component │ → │ Preview │
│              │    │           │    │ CSS          │    │ (Scripts 추가)   │    │         │
└─────────────┘    └───────────┘    └──────────────┘    └─────────────────┘    └─────────┘
       │                 │                 │                     │                   │
       ▼                 ▼                 ▼                     ▼                   ▼
   디자이너 작업      MCP로 추출      Figma_Conversion      RNBT_architecture     독립 검증
```

---

## 2. End-to-End 워크플로우 상세

### 2.1 Phase 1: Figma Design → Static HTML/CSS

**도구**: Figma MCP (Model Context Protocol)

**과정**:
1. 사용자가 Figma 링크(node-id 포함) 제공
2. `get_design_context` 호출 → 디자인 정보 + 에셋 추출
3. `get_screenshot` 호출 → 원본 스크린샷 (검증 기준)
4. Tailwind React → 순수 HTML/CSS 변환
5. Playwright 스크린샷 비교 → 픽셀 정확도 검증

**산출물 위치**: `Figma_Conversion/Conversion/[ProjectName]/[ComponentName]/`

```
time-trend-chart/
├── assets/                    # SVG, 이미지 에셋
├── screenshots/impl.png       # Playwright 검증 스크린샷
├── time-trend-chart.html      # 정적 HTML
└── time-trend-chart.css       # 순수 CSS
```

**핵심 원칙**:
- MCP 제공 값을 정확히 사용 (추측 금지)
- 순수 출력: HTML/CSS/에셋만 (스크립트 없음)
- 에셋은 반드시 `dirForAssetWrites`로 다운로드

### 2.2 Phase 2: Static HTML/CSS → Dynamic Component

**목적**: 정적 마크업에 스크립트를 추가하여 동적 컴포넌트로 변환

**과정**:
1. 정적 HTML을 `views/component.html`로 이동
2. 반복 요소를 `<template>` 패턴으로 변환
3. `scripts/register.js` 작성 (구독, 이벤트, 렌더링)
4. `scripts/destroy.js` 작성 (정리 로직)
5. `preview.html` 생성 (독립 검증 환경)

**산출물 위치**: `RNBT_architecture/Projects/[ProjectName]/page/components/[ComponentName]/`

```
TimeTrendChart/
├── views/component.html       # 동적 템플릿
├── styles/component.css       # 컴포넌트 CSS
├── scripts/
│   ├── register.js            # 초기화 + 구독
│   └── destroy.js             # 정리
├── assets/                    # 에셋 복사
├── preview.html               # 독립 검증
└── preview_screenshot.png     # 검증 결과
```

---

## 3. 컴포넌트 독립성의 설계 원리

### 3.1 왜 컴포넌트 독립성이 중요한가?

전통적인 웹 개발에서는 컴포넌트가 페이지에 종속됩니다:
- 페이지가 데이터를 가져와 컴포넌트에 전달
- 페이지 구조가 변경되면 컴포넌트도 영향받음
- 컴포넌트 단독 테스트 불가

**독립적 컴포넌트의 장점**:
- 페이지 개발 전에 컴포넌트 개발/검증 가능
- 다른 페이지에서 재사용 용이
- 병렬 개발 가능 (페이지 팀 ↔ 컴포넌트 팀)

### 3.2 Contract 기반 분리

컴포넌트와 페이지는 **Contract(약속)**를 통해 분리됩니다:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Contract (약속)                                                     │
│                                                                      │
│  1. Topic: 컴포넌트가 구독할 데이터 채널                              │
│     예: 'timeTrendData', 'institutionDelayData'                     │
│                                                                      │
│  2. Event: 컴포넌트가 발행할 사용자 상호작용 이벤트                   │
│     예: '@filterClicked', '@rowClicked'                             │
│                                                                      │
│  3. Data Structure: 컴포넌트가 기대하는 데이터 형태                   │
│     예: { series: [...], labels: [...] }                            │
└─────────────────────────────────────────────────────────────────────┘
```

**Contract만 합의되면**:
- 컴포넌트는 페이지 없이 mock 데이터로 개발
- 페이지는 컴포넌트 없이 데이터 흐름 설계
- 통합 시점에 연결만 하면 됨

### 3.3 Bottom-Up 접근법

페이지 설계가 없는 상황에서 컴포넌트가 먼저 Contract를 제안하는 방식:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Bottom-Up: Figma 디자인에서 데이터 구조 추론                         │
│                                                                      │
│  Figma 분석:                                                         │
│  - 반복 패턴 → 배열                                                  │
│  - 라벨+값 쌍 → 객체 필드                                            │
│  - 헤더 영역 → summary 객체                                          │
│                                                                      │
│  예: "구간별 성능현황" 컴포넌트                                       │
│  ┌─────────────────────────────────┐                                │
│  │ 전체: 681건 / 16,959.0 / 0.603 │ → header 객체                   │
│  ├─────────────────────────────────┤                                │
│  │ [실린더] 전체  61.0  0.001     │                                │
│  │ [실린더] WEB   32.5  0.002     │ → items 배열                    │
│  │ ...                            │                                │
│  └─────────────────────────────────┘                                │
│                                                                      │
│  추론된 구조:                                                         │
│  {                                                                   │
│    header: { count: 681, totalTps: 16959.0, avgResponse: 0.603 },   │
│    items: [                                                          │
│      { label: '전체', tps: 61.0, response: 0.001, cylinderType: 'green' } │
│    ]                                                                 │
│  }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

**Bottom-Up의 장점**:
- 페이지 설계 완료를 기다리지 않고 작업 시작 가능
- 실제 UI에서 필요한 데이터만 정의 (과잉 설계 방지)
- 컴포넌트 재사용성 극대화

### 3.4 Preview 검증

컴포넌트는 `preview.html`을 통해 페이지 없이 독립 검증됩니다:

```html
<!-- preview.html 구조 -->
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="styles/component.css">
</head>
<body>
    <!-- 컨테이너: Figma 선택 요소 크기 -->
    <div id="component-container">
        <!-- component.html 내용 -->
    </div>

    <!-- Mock 데이터로 검증 -->
    <script>
        const mockData = { /* 추론된 데이터 구조 */ };

        // GlobalDataPublisher 시뮬레이션
        GlobalDataPublisher.publish('topicName', { data: mockData });
    </script>
</body>
</html>
```

**검증 항목**:
- 렌더링 정확성 (스크린샷 비교)
- 데이터 바인딩 동작
- 이벤트 발생 확인
- 정리 로직 검증

---

## 4. 페이지와 컴포넌트의 역할 분리

### 4.1 역할 정의

| 영역 | 역할 | 책임 |
|------|------|------|
| **Page** | Orchestrator (지휘자) | 데이터 발행, 이벤트 핸들링, 컴포넌트 조율 |
| **Component** | Renderer (연주자) | 데이터 렌더링, 사용자 상호작용, 이벤트 발생 |

### 4.2 Page의 역할 상세

```javascript
// Page - before_load.js (컴포넌트 생성 전)
// 이벤트 핸들러 등록
this.eventBusHandlers = {
    '@filterClicked': async ({ event, targetInstance }) => {
        // 컴포넌트에서 발생한 이벤트 처리
        const filter = event.target.dataset.filter;
        GlobalDataPublisher.fetchAndPublish('timeTrendData', this, { filter });
    }
};
WKit.onEventBusHandlers(this.eventBusHandlers);

// Page - loaded.js (모든 컴포넌트 완료 후)
// 데이터 매핑 등록 및 발행
this.globalDataMappings = [
    {
        topic: 'timeTrendData',
        datasetInfo: { datasetName: 'api', param: { type: 'trend' } }
    }
];

fx.go(
    this.globalDataMappings,
    fx.each(GlobalDataPublisher.registerMapping),
    fx.each(({ topic }) => GlobalDataPublisher.fetchAndPublish(topic, this))
);
```

**Page가 하는 일**:
- 어떤 Topic에 어떤 API를 연결할지 결정
- 컴포넌트 이벤트를 받아 비즈니스 로직 실행
- 컴포넌트 간 통신 중재

**Page가 하지 않는 일**:
- 컴포넌트 내부 DOM 조작
- 렌더링 로직 구현
- 데이터 포맷팅

### 4.3 Component의 역할 상세

```javascript
// Component - register.js
// 1. 구독 등록
this.subscriptions = {
    timeTrendData: ['renderChart']
};

fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => GlobalDataPublisher.subscribe(topic, this, this[fn]), fnList)
    )
);

// 2. 이벤트 바인딩
this.customEvents = {
    click: {
        '.btn': '@filterClicked'
    }
};
WKit.bindEvents(this, this.customEvents);

// 3. 렌더링 함수
function renderChart(response) {
    const { data } = response;
    if (!data) return;  // Guard clause

    // 렌더링 로직
}
```

**Component가 하는 일**:
- 특정 Topic 구독 및 데이터 렌더링
- 사용자 상호작용을 이벤트로 발행
- 자신의 상태/리소스 관리

**Component가 하지 않는 일**:
- API 직접 호출
- 다른 컴포넌트 참조/조작
- 페이지 레벨 상태 관리

### 4.4 데이터 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Page (Orchestrator)                        │
│                                                                      │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │ before_load     │ → │ [Components     │ → │ loaded          │  │
│  │                 │    │  register]      │    │                 │  │
│  │ - 이벤트 핸들러 │    │                 │    │ - 데이터 발행   │  │
│  │   등록          │    │                 │    │ - Interval 설정 │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘  │
│           │                     │                      │             │
│           ▼                     ▼                      ▼             │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    GlobalDataPublisher                          ││
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐                   ││
│  │  │ Topic A   │  │ Topic B   │  │ Topic C   │                   ││
│  │  │ (trend)   │  │ (delay)   │  │ (status)  │                   ││
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                   ││
│  └────────┼──────────────┼──────────────┼──────────────────────────┘│
└───────────┼──────────────┼──────────────┼───────────────────────────┘
            │              │              │
            ▼              ▼              ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ TimeTrendChart│  │ InstitutionDelay│ │ PerformanceStatus│
│               │  │ Top5          │  │               │
│ - subscribe   │  │ - subscribe   │  │ - subscribe   │
│ - render      │  │ - render      │  │ - render      │
│ - emit event  │  │ - emit event  │  │ - emit event  │
└───────────────┘  └───────────────┘  └───────────────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                    WEventBus
                          │
                          ▼
                Page eventBusHandlers
```

---

## 5. 컴포넌트 라이프사이클

### 5.1 라이프사이클 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│  Component Lifecycle                                                 │
│                                                                      │
│  register.js                              destroy.js                 │
│  ┌─────────────────────────┐              ┌─────────────────────────┐│
│  │ 1. 라이브러리 초기화    │              │ 1. 구독 해제            ││
│  │ 2. 구독 등록            │              │ 2. 이벤트 제거          ││
│  │ 3. 이벤트 바인딩        │              │ 3. 라이브러리 정리      ││
│  │ 4. 렌더링 함수 바인딩   │              │ 4. 참조 null 처리       ││
│  └─────────────────────────┘              └─────────────────────────┘│
│              │                                        ▲              │
│              ▼                                        │              │
│         [Component Active]                            │              │
│              │                                        │              │
│              └───────── User Interaction ─────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 register.js 상세

```javascript
/*
 * Component - register.js
 *
 * 실행 시점: 컴포넌트 DOM이 생성된 직후
 * 컨텍스트: this = 컴포넌트 인스턴스
 */

// ============================================
// 1. 외부 라이브러리 초기화 (해당 시)
// ============================================
const chartContainer = this.element.querySelector('#echarts');
this.chartInstance = echarts.init(chartContainer);

// ResizeObserver로 반응형 처리
this.resizeObserver = new ResizeObserver(() => {
    this.chartInstance && this.chartInstance.resize();
});
this.resizeObserver.observe(chartContainer);

// ============================================
// 2. 구독 등록
// ============================================
this.subscriptions = {
    topicName: ['renderMethod']
};

this.renderMethod = renderMethod.bind(this);

fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => GlobalDataPublisher.subscribe(topic, this, this[fn]), fnList)
    )
);

// ============================================
// 3. 이벤트 바인딩
// ============================================
this.customEvents = {
    click: {
        '.selector': '@eventName'
    }
};

WKit.bindEvents(this, this.customEvents);

// ============================================
// 4. 렌더링 함수
// ============================================
function renderMethod(response) {
    const { data } = response;
    if (!data) return;  // Guard clause

    // 렌더링 로직
}
```

### 5.3 destroy.js 상세

```javascript
/*
 * Component - destroy.js
 *
 * 실행 시점: 컴포넌트가 제거되기 직전
 * 원칙: register에서 생성한 것은 destroy에서 정리 (대칭)
 */

// ============================================
// 1. 구독 해제
// ============================================
fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, _]) => GlobalDataPublisher.unsubscribe(topic, this))
);

// ============================================
// 2. 이벤트 제거
// ============================================
WKit.removeCustomEvents(this, this.customEvents);

// ============================================
// 3. 외부 라이브러리 정리
// ============================================
if (this.resizeObserver) {
    this.resizeObserver.disconnect();
    this.resizeObserver = null;
}

if (this.chartInstance) {
    this.chartInstance.dispose();
    this.chartInstance = null;
}

// ============================================
// 4. 참조 정리
// ============================================
this.renderMethod = null;
```

### 5.4 대칭 원칙

| register.js | destroy.js |
|-------------|------------|
| `subscribe(topic, this, fn)` | `unsubscribe(topic, this)` |
| `bindEvents(this, events)` | `removeCustomEvents(this, events)` |
| `echarts.init()` | `chartInstance.dispose()` |
| `new Tabulator()` | `tableInstance.destroy()` |
| `new ResizeObserver()` | `resizeObserver.disconnect()` |

---

## 6. 외부 라이브러리 통합 패턴

### 6.1 ECharts 통합

**초기화 패턴**:

```javascript
// register.js
const chartContainer = this.element.querySelector('#echarts');
this.chartInstance = echarts.init(chartContainer, null, {
    renderer: 'canvas'
});

// ResizeObserver로 컨테이너 크기 변화 감지
this.resizeObserver = new ResizeObserver(() => {
    this.chartInstance && this.chartInstance.resize();
});
this.resizeObserver.observe(chartContainer);
```

**렌더링 패턴**:

```javascript
function renderChart(response) {
    const { data } = response;
    if (!data) return;

    const option = {
        xAxis: { ... },
        yAxis: { ... },
        series: [ ... ]
    };

    try {
        this.chartInstance.setOption(option);
    } catch (e) {
        console.error('[Chart] setOption error:', e);
    }
}
```

**정리 패턴**:

```javascript
// destroy.js
if (this.resizeObserver) {
    this.resizeObserver.disconnect();
    this.resizeObserver = null;
}

if (this.chartInstance) {
    this.chartInstance.dispose();
    this.chartInstance = null;
}
```

### 6.2 Tabulator 통합

**초기화 패턴**:

```javascript
// register.js
const tableContainer = this.element.querySelector('.table-container');
const uniqueId = `tabulator-${this.id}`;
tableContainer.id = uniqueId;

this.tableInstance = new Tabulator(`#${uniqueId}`, {
    layout: 'fitColumns',
    height: '100%',
    rowHeight: 28,
    headerSort: false,
    columns: [
        { title: '기관명', field: 'name', ... },
        { title: '지연', field: 'delay', ... }
    ],
    rowClick: (e, row) => {
        WEventBus.emit('@rowClicked', { event: e, data: row.getData() });
    }
});
```

**렌더링 패턴**:

```javascript
function renderData(response) {
    const { data } = response;
    if (!data || !data.items) return;

    this.tableInstance.setData(data.items);
}
```

**정리 패턴**:

```javascript
// destroy.js
if (this.tableInstance) {
    this.tableInstance.destroy();
    this.tableInstance = null;
}
```

**커스텀 스타일링 주의사항**:

```css
/* Tabulator 기본 스타일 오버라이드 */
.tabulator .tabulator-table {
    background: transparent;  /* 기본 white 제거 */
}

.tabulator .tabulator-row {
    min-height: 28px;
    height: 28px;
}

.tabulator .tabulator-col-title {
    padding-right: 0 !important;  /* 헤더 텍스트 중앙 정렬 */
}
```

---

## 7. 데이터 키 미지정 문제 해결 (차트 재사용성)

### 7.1 문제 정의

차트 컴포넌트를 개발할 때 흔히 마주치는 문제:

```
문제: API 필드명을 미리 알 수 없음

API A: { time: "00", max_value: 100, min_value: 50 }
API B: { tm: "00", val_max: 100, val_min: 50 }
API C: { hour: "00", peak: 100, low: 50 }

→ 각 API마다 다른 차트 컴포넌트를 만들어야 하나?
```

### 7.2 해결 방법: Config 분리 패턴

**핵심 아이디어**: 필드 매핑을 config 객체로 분리하여 렌더링 로직과 데이터 구조를 독립시킴

```javascript
// register.js

// ============================================
// CONFIG: 데이터 구조 매핑 (정적 선언)
// ============================================
const config = {
    // X축에 사용할 API 필드
    xKey: 'tm',

    // 시리즈 정의: API 필드명 → 표시명 매핑
    seriesMap: [
        { key: 'val_max', name: '역대픽', color: '#526FE5' },
        { key: 'val_year', name: '연중최고픽', color: '#52BEE5' },
        { key: 'val_month', name: '월픽', color: '#009178' },
        { key: 'val_prev', name: '전일', color: '#52E5C3' },
        { key: 'val_today', name: '금일', color: '#AAFD84' }
    ],

    // 시리즈 공통 스타일
    smooth: true,
    symbol: 'none',
    areaStyle: true
};

// ============================================
// 렌더링 함수: config를 받아 범용적으로 동작
// ============================================
function renderLineData(config, response) {
    const { data } = response;
    if (!data) return;

    const { xKey, seriesMap, smooth, symbol, areaStyle } = config;

    const option = {
        xAxis: {
            type: 'category',
            // config.xKey로 X축 데이터 추출
            data: fx.go(data, fx.map(d => d[xKey]))
        },
        series: fx.go(
            seriesMap,
            fx.map(s => ({
                name: s.name,
                type: 'line',
                smooth,
                symbol,
                // config.seriesMap[].key로 각 시리즈 데이터 추출
                data: fx.go(data, fx.map(d => d[s.key])),
                itemStyle: { color: s.color },
                lineStyle: { color: s.color }
            }))
        )
    };

    this.chartInstance.setOption(option);
}

// ============================================
// curry로 config 바인딩
// ============================================
this.renderChart = fx.curry(renderLineData)(config).bind(this);
```

### 7.3 효과

**재사용 시나리오**:

```javascript
// 같은 컴포넌트, 다른 config로 다양한 API 대응

// API A용 config
const configA = {
    xKey: 'time',
    seriesMap: [
        { key: 'max_value', name: '최대', color: '#FF0000' },
        { key: 'min_value', name: '최소', color: '#0000FF' }
    ]
};

// API B용 config
const configB = {
    xKey: 'hour',
    seriesMap: [
        { key: 'peak', name: 'Peak', color: '#00FF00' },
        { key: 'avg', name: 'Average', color: '#FFFF00' }
    ]
};

// 동일한 renderLineData 함수 재사용!
```

**장점**:
- 렌더링 로직 1회 작성으로 다양한 데이터 소스 대응
- API 변경 시 config만 수정 (로직 변경 불필요)
- 타입별 시리즈 색상, 스타일 일괄 관리

### 7.4 curry 패턴 활용

```javascript
// fx.curry를 사용한 config 바인딩

// 1. 일반 함수 정의 (config를 첫 번째 인자로)
function renderLineData(config, response) { ... }

// 2. curry로 부분 적용
const renderWithConfig = fx.curry(renderLineData)(config);

// 3. 인스턴스에 바인딩
this.renderChart = renderWithConfig.bind(this);

// 4. GlobalDataPublisher가 response만 전달
// → renderChart(response) 호출 시 config는 이미 바인딩됨
```

---

## 8. 실제 적용 사례

### 8.1 TimeTrendChart (ECharts Area Chart)

| 항목 | 내용 |
|------|------|
| **Topic** | `timeTrendData` |
| **Event** | `@filterClicked` |
| **라이브러리** | ECharts |
| **크기** | 831 × 350px |

**데이터 구조**:
```javascript
[
  { tm: "00", val_max: 1200, val_year: 1100, val_month: 600, val_prev: 400, val_today: 300 },
  { tm: "02", val_max: 1300, val_year: 1150, val_month: 650, val_prev: 420, val_today: 320 },
  ...
]
```

**Config 패턴 적용**:
```javascript
const config = {
    xKey: 'tm',
    seriesMap: [
        { key: 'val_max', name: '역대픽', color: '#526FE5' },
        { key: 'val_year', name: '연중최고픽', color: '#52BEE5' },
        { key: 'val_month', name: '월픽', color: '#009178' },
        { key: 'val_prev', name: '전일', color: '#52E5C3' },
        { key: 'val_today', name: '금일', color: '#AAFD84' }
    ],
    yAxis: { min: 0, max: 1800, interval: 600 }
};
```

### 8.2 InstitutionDelayTop5Tabulator (Tabulator Table)

| 항목 | 내용 |
|------|------|
| **Topic** | `institutionDelayData` |
| **Event** | `@rowClicked` |
| **라이브러리** | Tabulator |
| **크기** | 478 × 254px |

**데이터 구조**:
```javascript
{
  items: [
    { category: 'electronic', name: '국민은행', timeout: 5, delay: 3, inquiry: 10, deposit: 8 },
    { category: 'openbanking', name: '카카오뱅크', timeout: 2, delay: 1, inquiry: 5, deposit: 3 }
  ]
}
```

**Tabulator 컬럼 정의**:
```javascript
columns: [
    { title: '', field: 'category', width: 40, formatter: categoryFormatter },
    { title: '기관명', field: 'name', width: 93 },
    { title: 'time_out', field: 'timeout', formatter: numberFormatter },
    { title: '지연', field: 'delay', formatter: numberFormatter },
    { title: '조회대행', field: 'inquiry', formatter: numberFormatter },
    { title: '입지대행', field: 'deposit', formatter: numberFormatter }
]
```

### 8.3 PerformanceStatus (Vanilla HTML)

| 항목 | 내용 |
|------|------|
| **Topic** | `performanceStatusData` |
| **Event** | `@itemClicked` |
| **라이브러리** | 없음 (Vanilla) |
| **크기** | 524 × 350px |

**데이터 구조**:
```javascript
{
  header: { count: 681, totalTps: 16959.0, avgResponse: 0.603 },
  items: [
    { label: '전체', number: 681, tps: 61.0, response: 0.001, cylinderType: 'green' },
    { label: 'WEB', number: 150, tps: 32.5, response: 0.002, cylinderType: 'blue' }
  ]
}
```

**Template 패턴 적용**:
```html
<template id="item-template">
    <div class="item">
        <div class="cylinder-img"><img src="" alt=""></div>
        <div class="item-number"></div>
        <div class="item-label"><p></p></div>
        <div class="item-tps"><p></p></div>
        <div class="item-response"><p></p></div>
    </div>
</template>
```

### 8.4 BusinessStatus (Vanilla HTML)

| 항목 | 내용 |
|------|------|
| **Topic** | `businessStatusData` |
| **Event** | `@rowClicked` |
| **라이브러리** | 없음 (Vanilla) |
| **크기** | 459 × 305px |

**데이터 구조**:
```javascript
{
  sections: [
    {
      titles: ['1Q', '상품처리'],
      rows: [
        { label: '동시 사용자 수', col1: { current: '10,906', peak: '18,328' }, col2: { current: '2,425', peak: '3,992' } }
      ],
      colorType: 'cyan'
    }
  ]
}
```

---

## 9. 폴더 구조 및 파일 규약

### 9.1 Figma_Conversion 산출물

```
Figma_Conversion/
└── Conversion/
    └── [ProjectName]/
        └── [ComponentName]/
            ├── assets/                # SVG, 이미지 에셋
            ├── screenshots/
            │   └── impl.png           # Playwright 검증 스크린샷
            ├── [component-name].html  # 정적 HTML
            └── [component-name].css   # 순수 CSS
```

### 9.2 RNBT_architecture 컴포넌트

```
RNBT_architecture/
└── Projects/
    └── [ProjectName]/
        └── page/
            └── components/
                └── [ComponentName]/
                    ├── views/
                    │   └── component.html     # 동적 템플릿
                    ├── styles/
                    │   └── component.css      # 컴포넌트 CSS
                    ├── scripts/
                    │   ├── register.js        # 초기화 + 구독
                    │   └── destroy.js         # 정리
                    ├── assets/                # 에셋 복사
                    ├── preview.html           # 독립 검증
                    └── preview_screenshot.png # 검증 결과
```

### 9.3 파일별 역할

| 파일 | 역할 |
|------|------|
| `views/component.html` | 동적 렌더링을 위한 HTML 템플릿 (`<template>` 포함) |
| `styles/component.css` | 컴포넌트 스타일 (CSS nesting 구조) |
| `scripts/register.js` | 구독, 이벤트 바인딩, 라이브러리 초기화 |
| `scripts/destroy.js` | 구독 해제, 이벤트 제거, 라이브러리 정리 |
| `preview.html` | 독립 검증 환경 (mock 데이터 포함) |
| `assets/` | SVG, 이미지 등 정적 에셋 |

### 9.4 register.js JSDoc 헤더 규약

```javascript
/*
 * Page - [ComponentName] Component - register
 * [컴포넌트 설명]
 *
 * Subscribes to: [topic명]
 * Events: [@이벤트명]
 *
 * 데이터 구조 (Bottom-Up 추론):
 * {
 *   field1: type,
 *   field2: type,
 *   items: [
 *     { ... }
 *   ]
 * }
 */
```

---

## 10. 요약

### 10.1 핵심 설계 원칙

1. **Contract 기반 분리**: Topic과 Event로 페이지-컴포넌트 인터페이스 정의
2. **Bottom-Up 접근**: Figma 디자인에서 데이터 구조 추론
3. **Preview 검증**: 페이지 없이 컴포넌트 독립 테스트
4. **대칭 원칙**: register에서 생성한 것은 destroy에서 정리
5. **Config 분리**: 데이터 키를 config로 외부화하여 재사용성 확보

### 10.2 워크플로우 체크리스트

- [ ] Figma MCP로 디자인 정보 추출
- [ ] 정적 HTML/CSS 생성 및 Playwright 검증
- [ ] 데이터 구조 추론 (Bottom-Up)
- [ ] Topic/Event Contract 정의
- [ ] component.html 작성 (template 패턴)
- [ ] register.js 작성 (구독, 이벤트, 렌더링)
- [ ] destroy.js 작성 (정리 로직)
- [ ] preview.html 작성 (mock 데이터)
- [ ] 스크린샷 비교 검증

---

**문서 버전**: 1.0.0
**작성일**: 2025-12-08
**작성자**: Claude Code
