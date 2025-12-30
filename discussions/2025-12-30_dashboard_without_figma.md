# Figma 없이 대시보드 만들기

## 배경

Figma 디자인이 없는 상황에서도 대시보드 프로토타입이 필요한 경우:

- 기술 검증 (PoC)
- 내부 데모
- API 설계 전 UI 프로토타이핑
- 컴포넌트 라이브러리 구축

---

## 핵심 원칙

**컴포넌트 우선 개발 (Component-First)**

```
Phase 1: 독립 컴포넌트 개발 (API 없이)
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Chart   │  │ Table   │  │ Card    │  ← Mock 데이터 + preview.html
└─────────┘  └─────────┘  └─────────┘
      │            │            │
      └────────────┼────────────┘
                   ▼
Phase 2: 페이지 통합 (API 연동)
┌─────────────────────────────────────┐
│              Page                    │
│  ┌───────┐ ┌───────┐ ┌───────┐     │ ← page_scripts + API
│  │ Chart │ │ Table │ │ Card  │     │
│  └───────┘ └───────┘ └───────┘     │
└─────────────────────────────────────┘
```

---

## 컴포넌트 구조

```
Projects/[프로젝트]/page/components/SensorCard/
├── views/component.html
├── styles/component.css
├── scripts/
│   ├── register.js       ← API 미정이어도 완성 가능
│   └── beforeDestroy.js  ← API 미정이어도 완성 가능
└── preview.html          ← 브라우저에서 직접 테스트
```

---

## register.js 템플릿

TempHumiditySensor 패턴 기반:

```javascript
/**
 * SensorCard - register.js
 */

const { bind3DEvents, fetchData } = WKit;

initComponent.call(this);

function initComponent() {
    // ======================
    // DATA DEFINITION (API 확정 시 수정)
    // ======================
    const assetId = this.setter.assetInfo?.assetId || 'sensor-001';

    this.datasetInfo = [
        { datasetName: 'sensor', param: { id: assetId }, render: ['renderSensorInfo'] }
    ];

    // ======================
    // DATA CONFIG (API 확정 시 수정)
    // ======================
    this.sensorInfoConfig = [
        { key: 'temperature', selector: '.temp' },
        { key: 'humidity', selector: '.humidity' },
        { key: 'status', selector: '.status', dataAttr: 'status' }
    ];

    // ======================
    // RENDER FUNCTIONS
    // ======================
    this.renderSensorInfo = renderSensorInfo.bind(this);

    // ======================
    // CUSTOM EVENTS
    // ======================
    this.customEvents = {
        click: '@sensorCardClicked'
    };

    bind3DEvents(this, this.customEvents);

    console.log('[SensorCard] Registered:', assetId);
}

// ======================
// RENDER FUNCTIONS
// ======================
function renderSensorInfo(data) {
    fx.go(
        this.sensorInfoConfig,
        fx.each(({ key, selector, dataAttr }) => {
            const el = this.element.querySelector(selector);
            if (el) {
                el.textContent = data[key];
                if (dataAttr) el.dataset[dataAttr] = data[key];
            }
        })
    );
}

// ======================
// PUBLIC METHODS
// ======================
this.loadData = function() {
    fx.go(
        this.datasetInfo,
        fx.each(({ datasetName, param, render }) =>
            fx.go(
                fetchData(this.page, datasetName, param),
                result => result?.response?.data,
                data => data && render.forEach(fn => this[fn](data))
            )
        )
    );
};
```

---

## beforeDestroy.js 템플릿

```javascript
/**
 * SensorCard - beforeDestroy.js
 */

// Popup Mixin 사용 시
if (this.destroyPopup) {
    this.destroyPopup();
}

console.log('[SensorCard] Destroyed:', this.setter?.assetInfo?.assetId);
```

---

## 차트 컴포넌트 예시

TempHumiditySensor 차트 패턴 기반:

```javascript
// TrendChart/scripts/register.js

const { bind3DEvents, fetchData } = WKit;
const { applyShadowPopupMixin, applyEChartsMixin } = PopupMixin;

initComponent.call(this);

function initComponent() {
    // ======================
    // DATA DEFINITION
    // ======================
    this.datasetInfo = [
        { datasetName: 'chartData', param: {}, render: ['renderChart'] }
    ];

    // ======================
    // CHART CONFIG
    // ======================
    this.chartConfig = {
        xKey: 'timestamps',
        series: [
            { yKey: 'values', name: 'Value', color: '#3b82f6', yAxisIndex: 0 }
        ],
        yAxis: [
            { name: '', position: 'left' }
        ]
    };

    // ======================
    // RENDER FUNCTIONS
    // ======================
    this.renderChart = renderChart.bind(this);

    // ======================
    // MIXIN SETUP
    // ======================
    applyEChartsMixin(this);

    console.log('[TrendChart] Registered');
}

function renderChart(data) {
    const option = buildChartOption(this.chartConfig, data);
    this.updateChart('.chart-container', option);
}

function buildChartOption(config, data) {
    const { xKey, series: seriesConfig, yAxis: yAxisConfig } = config;

    return {
        xAxis: { type: 'category', data: data[xKey] },
        yAxis: yAxisConfig.map(axis => ({
            type: 'value',
            name: axis.name,
            position: axis.position
        })),
        series: seriesConfig.map(({ yKey, name, color, yAxisIndex }) => ({
            name, type: 'line', yAxisIndex,
            data: data[yKey],
            lineStyle: { color }
        }))
    };
}
```

```javascript
// TrendChart/scripts/beforeDestroy.js

if (this.destroyChart) {
    this.destroyChart();
}
```

---

## 패턴 규칙

### 이벤트 바인딩

```javascript
// ✓ DO: WKit의 bind3DEvents 사용
const { bind3DEvents } = WKit;

this.customEvents = {
    click: '@componentClicked',
    dblclick: '@componentDblClicked'
};

bind3DEvents(this, this.customEvents);
```

### 데이터 페칭

```javascript
// ✓ DO: fetchData + fx.go 패턴
const { fetchData } = WKit;

fx.go(
    fetchData(this.page, datasetName, param),
    result => result?.response?.data,
    data => data && this.renderData(data)
);
```

### Config 패턴

API 필드명이 바뀌면 config만 수정:

```javascript
// 변경 전: { temperature: 24.5 }
// 변경 후: { temp_value: 24.5 }

this.sensorInfoConfig = [
    { key: 'temp_value', selector: '.temp' },  // key만 변경
];
```

### 함수 바인딩

```javascript
// ✓ DO: bind(this) 패턴
this.renderSensorInfo = renderSensorInfo.bind(this);
this.renderChart = renderChart.bind(this);

// 외부 함수로 정의
function renderSensorInfo(data) {
    // this가 컴포넌트 인스턴스를 가리킴
}
```

---

## 완성도 체크

| 항목 | API 미정 | API 확정 후 |
|------|---------|------------|
| register.js | ✓ 완성 | datasetInfo, config만 수정 |
| beforeDestroy.js | ✓ 완성 | 변경 없음 |
| preview.html | ✓ Mock 테스트 | 변경 없음 |
| customEvents | ✓ 완성 | 변경 없음 |
| datasetInfo | ✓ placeholder | datasetName/param 확정 |

---

## 체크리스트

### 컴포넌트 개발 시

- [ ] `preview.html`로 독립 실행 가능한가?
- [ ] Mock 데이터로 모든 상태 테스트 가능한가?
- [ ] `bind3DEvents`로 이벤트 바인딩했는가?
- [ ] `customEvents` 객체로 이벤트 정의했는가?
- [ ] `beforeDestroy.js`에서 정리 로직 완성했는가?

### 페이지 통합 시

- [ ] `datasetInfo`의 datasetName이 API와 일치하는가?
- [ ] config의 key가 API 응답 필드와 일치하는가?

---

*작성일: 2025-12-30*
