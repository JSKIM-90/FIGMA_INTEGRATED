---
name: create-3d-self-contained-component
description: 3D 환경에서 사용하는 자기완결 컴포넌트를 생성합니다. 페이지에 의존하지 않고 스스로 데이터를 fetch하며, Shadow DOM 팝업을 사용합니다. Use when creating 3D asset components, self-contained components with datasetInfo, or components with Shadow DOM popups.
---

# 3D 자기완결 컴포넌트 생성

3D 환경에서 사용하는 **자기완결 컴포넌트**를 생성하는 Skill입니다.
페이지 오케스트레이션 없이 컴포넌트가 직접 데이터를 fetch합니다.
Figma MCP는 필요하지 않습니다.

---

## 일반 컴포넌트 vs 자기완결 컴포넌트

| 구분 | 일반 컴포넌트 | 자기완결 컴포넌트 |
|------|--------------|------------------|
| **데이터 소스** | 페이지에서 발행 (GlobalDataPublisher) | 컴포넌트가 직접 fetch (WKit.fetchData) |
| **구독 방식** | `this.subscriptions` | `this.datasetInfo` |
| **렌더링 트리거** | 토픽 발행 시 자동 호출 | Public Method 호출 시 fetch → render |
| **사용 환경** | 2D 대시보드 | 3D 씬 (클릭 이벤트 기반) |
| **팝업 방식** | 일반 DOM | Shadow DOM (스타일 격리) |

---

## 출력 구조

```
RNBT_architecture/Projects/[프로젝트명]/page/components/[ComponentName]/
├── views/component.html       # 3D 오브젝트 마크업 (최소한)
├── styles/component.css       # 3D 오브젝트 스타일 (최소한)
├── scripts/
│   ├── register.js            # datasetInfo + 팝업 + Public Methods
│   └── beforeDestroy.js       # destroyPopup 호출
└── preview.html               # Mock 데이터 + 팝업 테스트
```

---

## 핵심 개념

### 1. datasetInfo 패턴

페이지의 globalDataMappings 대신 컴포넌트가 직접 데이터 정의:

```javascript
// 페이지 오케스트레이션 방식 (일반 컴포넌트)
// page/page_scripts/loaded.js
this.globalDataMappings = [
    { topic: 'upsData', datasetName: 'ups', param: { id: 1 } }
];

// 자기완결 방식 (3D 컴포넌트)
// component/scripts/register.js
this.datasetInfo = [
    { datasetName: 'ups', param: { id: assetId }, render: ['renderUPSInfo'] },
    { datasetName: 'upsHistory', param: { id: assetId }, render: ['renderChart'] }
];
```

### 2. Public Methods 패턴

페이지에서 호출 가능한 메서드 노출:

```javascript
// 컴포넌트
this.showDetail = showDetail.bind(this);
this.hideDetail = hideDetail.bind(this);

// 페이지 eventBusHandler에서 호출
'@upsClicked': ({ source }) => {
    source.showDetail();  // 컴포넌트의 Public Method 직접 호출
}
```

### 3. Shadow DOM 팝업

스타일 격리를 위한 Shadow DOM 사용:

```javascript
applyShadowPopupMixin(this, {
    getHTML: () => extractTemplate(htmlCode, 'popup-ups'),
    getStyles: () => cssCode,
    onCreated: this.onPopupCreated
});
```

### 4. Template 기반 팝업 마크업

publishCode에서 template 태그로 팝업 HTML 제공:

```html
<template id="popup-ups">
    <div class="popup-overlay">
        <div class="popup-content">
            <!-- 팝업 내용 -->
        </div>
    </div>
</template>
```

---

## register.js 템플릿

```javascript
/**
 * [ComponentName] - Self-Contained 3D Component
 *
 * applyShadowPopupMixin을 사용한 자기 완결 컴포넌트
 *
 * 핵심 구조:
 * 1. datasetInfo - 데이터 정의
 * 2. Data Config - API 필드 매핑
 * 3. 렌더링 함수 바인딩
 * 4. Public Methods - Page에서 호출
 * 5. customEvents - 이벤트 발행
 * 6. Template Data - HTML/CSS (publishCode에서 로드)
 * 7. Popup - template 기반 Shadow DOM 팝업
 */

const { bind3DEvents, fetchData } = WKit;
const { applyShadowPopupMixin, applyEChartsMixin } = PopupMixin;

// ======================
// TEMPLATE HELPER
// ======================
function extractTemplate(htmlCode, templateId) {
    const parser = new DOMParser();
    const doc = parser.parseFromString(htmlCode, 'text/html');
    const template = doc.querySelector(`template#${templateId}`);
    return template?.innerHTML || '';
}

initComponent.call(this);

function initComponent() {
    // ======================
    // 1. 데이터 정의
    // ======================
    const assetId = this.setter.ecoAssetInfo.assetId;

    this.datasetInfo = [
        { datasetName: 'TBD_datasetName', param: { id: assetId }, render: ['renderInfo'] },
        { datasetName: 'TBD_historyDataset', param: { id: assetId }, render: ['renderChart'] }
    ];

    // ======================
    // 2. Data Config (API 필드 매핑)
    // ======================
    this.infoConfig = [
        { key: 'name', selector: '.item-name' },
        { key: 'status', selector: '.item-status', dataAttr: 'status' },
        { key: 'value', selector: '.item-value', suffix: '%' }
    ];

    this.chartConfig = {
        xKey: 'timestamps',
        series: [
            { yKey: 'value', name: 'Value', color: '#3b82f6', smooth: true }
        ],
        optionBuilder: getLineChartOption
    };

    // ======================
    // 3. 렌더링 함수 바인딩
    // ======================
    this.renderInfo = renderInfo.bind(this, this.infoConfig);
    this.renderChart = renderChart.bind(this, this.chartConfig);

    // ======================
    // 4. Public Methods
    // ======================
    this.showDetail = showDetail.bind(this);
    this.hideDetail = hideDetail.bind(this);

    // ======================
    // 5. 이벤트 발행
    // ======================
    this.customEvents = {
        click: '@TBD_componentClicked'
    };

    bind3DEvents(this, this.customEvents);

    // ======================
    // 6. Template Config
    // ======================
    this.templateConfig = {
        popup: 'popup-TBD_componentName',
    };

    // ======================
    // 7. Popup (template 기반)
    // ======================
    this.popupCreatedConfig = {
        chartSelector: '.chart-container',
        events: {
            click: {
                '.close-btn': () => this.hideDetail()
            }
        }
    };

    const { htmlCode, cssCode } = this.properties.publishCode || {};
    this.getPopupHTML = () => extractTemplate(htmlCode || '', this.templateConfig.popup);
    this.getPopupStyles = () => cssCode || '';
    this.onPopupCreated = onPopupCreated.bind(this, this.popupCreatedConfig);

    applyShadowPopupMixin(this, {
        getHTML: this.getPopupHTML,
        getStyles: this.getPopupStyles,
        onCreated: this.onPopupCreated
    });

    applyEChartsMixin(this);

    console.log('[ComponentName] Registered:', assetId);
}

// ======================
// PUBLIC METHODS
// ======================

function showDetail() {
    this.showPopup();
    fx.go(
        this.datasetInfo,
        fx.each(({ datasetName, param, render }) =>
            fx.go(
                fetchData(this.page, datasetName, param),
                result => result?.response?.data,
                data => data && render.forEach(fn => this[fn](data))
            )
        )
    ).catch(e => {
        console.error('[ComponentName]', e);
        this.hidePopup();
    });
}

function hideDetail() {
    this.hidePopup();
}

// ======================
// RENDER FUNCTIONS
// ======================

function renderInfo(config, data) {
    fx.go(
        config,
        fx.each(({ key, selector, dataAttr, suffix }) => {
            const el = this.popupQuery(selector);
            if (!el) return;
            const value = data[key];
            el.textContent = suffix ? `${value}${suffix}` : value;
            dataAttr && (el.dataset[dataAttr] = value);
        })
    );
}

function renderChart(config, data) {
    const { optionBuilder, ...chartConfig } = config;
    const option = optionBuilder(chartConfig, data);
    this.updateChart('.chart-container', option);
}

// ======================
// CHART OPTION BUILDER
// ======================

function getLineChartOption(config, data) {
    const { xKey, series: seriesConfig } = config;

    return {
        grid: { left: 45, right: 16, top: 30, bottom: 24 },
        legend: {
            data: seriesConfig.map(s => s.name),
            top: 0,
            textStyle: { color: '#8892a0', fontSize: 11 }
        },
        tooltip: {
            trigger: 'axis',
            backgroundColor: '#1a1f2e',
            borderColor: '#2a3142',
            textStyle: { color: '#e0e6ed', fontSize: 12 }
        },
        xAxis: {
            type: 'category',
            data: data[xKey],
            axisLine: { lineStyle: { color: '#333' } },
            axisLabel: { color: '#888', fontSize: 10 }
        },
        yAxis: {
            type: 'value',
            axisLine: { show: false },
            axisLabel: { color: '#888', fontSize: 10 },
            splitLine: { lineStyle: { color: '#333' } }
        },
        series: seriesConfig.map(({ yKey, name, color, smooth }) => ({
            name,
            type: 'line',
            data: data[yKey],
            smooth,
            symbol: 'none',
            lineStyle: { color, width: 2 }
        }))
    };
}

// ======================
// POPUP LIFECYCLE
// ======================

function onPopupCreated({ chartSelector, events }) {
    chartSelector && this.createChart(chartSelector);
    events && this.bindPopupEvents(events);
}
```

---

## beforeDestroy.js 템플릿

```javascript
/**
 * [ComponentName] - Destroy Script
 * 컴포넌트 정리 (Shadow DOM 팝업 + 차트)
 */

this.destroyPopup();
console.log('[ComponentName] Destroyed:', this.setter?.ecoAssetInfo?.assetId);
```

---

## Mixin API 참조

### applyShadowPopupMixin

Shadow DOM 기반 팝업 기능 주입:

```javascript
applyShadowPopupMixin(this, {
    getHTML: () => string,      // 팝업 HTML 반환
    getStyles: () => string,    // 팝업 CSS 반환
    onCreated: () => void       // 팝업 생성 후 콜백
});

// 주입되는 메서드
this.showPopup()                // 팝업 표시
this.hidePopup()                // 팝업 숨김
this.destroyPopup()             // 팝업 + 차트 정리
this.popupQuery(selector)       // Shadow DOM 내 요소 선택
this.popupQueryAll(selector)    // Shadow DOM 내 요소들 선택
this.bindPopupEvents(events)    // Shadow DOM 내 이벤트 바인딩
```

### applyEChartsMixin

ECharts 차트 기능 주입:

```javascript
applyEChartsMixin(this);

// 주입되는 메서드
this.createChart(selector)          // 차트 인스턴스 생성
this.updateChart(selector, option)  // 차트 옵션 업데이트
```

---

## 페이지 연동

### datasetList.json 등록

```json
{
    "ups": {
        "rest_api": "{\"method\":\"GET\",\"url\":\"http://localhost:3000/api/ups\"}"
    },
    "upsHistory": {
        "rest_api": "{\"method\":\"GET\",\"url\":\"http://localhost:3000/api/ups/history\"}"
    }
}
```

### 페이지 eventBusHandler

```javascript
// page/page_scripts/before_load.js
this.eventBusHandlers = {
    '@upsClicked': ({ source }) => {
        source.showDetail();  // Public Method 호출
    }
};

onEventBusHandlers(this.eventBusHandlers);
```

---

## 생성/정리 매칭 테이블

| 생성 | 정리 |
|------|------|
| `applyShadowPopupMixin(this, ...)` | `this.destroyPopup()` |
| `applyEChartsMixin(this)` | (destroyPopup 내부에서 처리) |
| `bind3DEvents(this, customEvents)` | (프레임워크가 처리) |

---

## TBD 패턴

API 명세 확정 전 개발:

```javascript
// datasetInfo
this.datasetInfo = [
    { datasetName: 'TBD_datasetName', param: { id: assetId }, render: ['renderInfo'] }
];

// Config
this.infoConfig = [
    { key: 'TBD_fieldName', selector: '.TBD-selector' }
];

// customEvents
this.customEvents = {
    click: '@TBD_componentClicked'
};

// templateConfig
this.templateConfig = {
    popup: 'popup-TBD_componentName'
};
```

---

## preview.html 구조

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>[ComponentName] - Preview</title>
    <script src="https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js"></script>
</head>
<body>
    <!-- 버튼으로 다양한 상태 테스트 -->
    <div class="preview-controls">
        <button onclick="showNormal()">Normal</button>
        <button onclick="showWarning()">Warning</button>
        <button onclick="showCritical()">Critical</button>
        <button onclick="destroyInstance()">Destroy</button>
    </div>

    <div id="popup-host"></div>

    <script>
        // TEMPLATE DATA (publishCode와 동일)
        const htmlCode = `<template id="popup-...">...</template>`;
        const cssCode = `/* Shadow DOM 스타일 */`;

        // MOCK IMPLEMENTATIONS
        const fx = { go: ..., each: ... };
        const WKit = { bind3DEvents: ..., fetchData: ... };
        const PopupMixin = { applyShadowPopupMixin: ..., applyEChartsMixin: ... };

        // MOCK DATA
        const mockData = { normal: {...}, warning: {...}, critical: {...} };
        let currentStatus = 'normal';

        // COMPONENT INSTANCE
        let instance = null;

        function createInstance() { ... }
        function showNormal() { currentStatus = 'normal'; instance?.showDetail(); }
        function showWarning() { currentStatus = 'warning'; instance?.showDetail(); }
        function showCritical() { currentStatus = 'critical'; instance?.showDetail(); }
        function destroyInstance() { ... }

        // register.js 로직 복사
        function initComponent() { ... }
        function showDetail() { ... }
        function renderInfo() { ... }
        function renderChart() { ... }
    </script>
</body>
</html>
```

---

## 금지 사항

```
❌ GlobalDataPublisher 구독 사용
- 자기완결 컴포넌트는 datasetInfo + fetchData 사용
- subscribe/unsubscribe 패턴 사용 금지

❌ 페이지에서 데이터 발행
- 컴포넌트가 직접 fetch
- 페이지는 이벤트 핸들러만 등록

❌ destroyPopup 누락
- beforeDestroy.js에서 반드시 호출
- 차트 인스턴스 메모리 누수 방지

❌ popupQuery 대신 document.querySelector
- Shadow DOM 내부는 popupQuery로만 접근
- document.querySelector는 Shadow DOM 내부 접근 불가
```

---

## 완료 체크리스트

```
- [ ] datasetInfo 정의 완료
    - [ ] datasetName 매핑
    - [ ] param 정의 (assetId 등)
    - [ ] render 함수 목록
- [ ] Data Config 정의 완료
    - [ ] infoConfig (필드 매핑)
    - [ ] chartConfig (차트 설정)
- [ ] Public Methods 정의 완료
    - [ ] showDetail (팝업 표시 + 데이터 fetch)
    - [ ] hideDetail (팝업 숨김)
- [ ] customEvents 정의 (3D 이벤트)
- [ ] templateConfig 정의 (팝업 template ID)
- [ ] popupCreatedConfig 정의
    - [ ] chartSelector
    - [ ] events (close-btn 등)
- [ ] Mixin 적용
    - [ ] applyShadowPopupMixin
    - [ ] applyEChartsMixin (차트 있는 경우)
- [ ] beforeDestroy.js 작성
    - [ ] destroyPopup 호출
- [ ] preview.html 작성
    - [ ] Mock data 정의
    - [ ] 다양한 상태 테스트 버튼
    - [ ] register.js 로직 복사
- [ ] 브라우저에서 preview.html 열어 확인
- [ ] datasetList.json에 API 등록
- [ ] 페이지 eventBusHandler에 이벤트 등록
```

---

## 참고 예제

- `RNBT_architecture/Projects/ECO/page/components/UPS/` - UPS 자기완결 컴포넌트
- `RNBT_architecture/Projects/ECO/datasetList.json` - API 엔드포인트
- `RNBT_architecture/Projects/ECO/page/page_scripts/before_load.js` - 이벤트 핸들러
