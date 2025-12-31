---
name: create-component
description: 정적 HTML/CSS를 RNBT 동적 컴포넌트로 변환합니다. Figma Conversion에서 생성된 정적 파일을 RNBT_architecture 패턴에 맞게 동적 컴포넌트로 변환합니다. Use when converting static HTML to dynamic components, creating RNBT components, or implementing components with data binding and event handling.
---

# RNBT 동적 컴포넌트 생성

정적 HTML/CSS를 **RNBT 동적 컴포넌트**로 변환하는 Skill입니다.
Figma MCP는 필요하지 않습니다.

---

## 입력

Figma Conversion에서 생성된 정적 파일:
```
Figma_Conversion/Conversion/[프로젝트명]/[컴포넌트명]/
├── assets/
├── [컴포넌트명].html
└── [컴포넌트명].css
```

## 출력

RNBT_architecture 동적 컴포넌트:
```
RNBT_architecture/Projects/[프로젝트명]/page/components/[ComponentName]/
├── assets/                    # SVG, 이미지 등 (Figma_Conversion에서 복사)
├── views/component.html       # 데이터 바인딩 마크업
├── styles/component.css       # 스타일 (.component-name 스코프)
├── scripts/
│   ├── register.js            # 초기화 + 메서드 정의
│   └── beforeDestroy.js       # 정리
└── preview.html               # 독립 테스트 (Mock 데이터 포함)
```

---

## 워크플로우

```
1. 정적 HTML 분석
   └─ Figma Conversion에서 생성된 HTML/CSS

2. 데이터 바인딩 마크업 변환
   <!-- 정적 -->
   <div class="value">1,234</div>

   <!-- 동적 -->
   <div class="value" data-bind="tps"></div>

3. register.js 작성
   - subscriptions 정의
   - customEvents 정의
   - Config 정의
   - 렌더 함수 바인딩

4. beforeDestroy.js 작성
   - unsubscribe
   - removeCustomEvents
   - 참조 제거

5. preview.html 작성
   - Mock 데이터로 독립 테스트
   - 서버 없이 브라우저에서 확인 가능
```

---

## 핵심 개념: 역할 분리

```
페이지 = 오케스트레이터
- 데이터 정의 (globalDataMappings)
- Interval 관리 (refreshIntervals)
- Param 관리 (currentParams)
- 이벤트 핸들러 등록 (eventBusHandlers)

컴포넌트 = 독립적 구독자
- 필요한 topic만 구독 (subscriptions)
- 이벤트 발행만 (@eventName)
- 데이터 렌더링만 집중
- 페이지의 내부 구조 몰라도 됨
```

---

## 라이프사이클

### 컴포넌트 라이프사이클

| 단계 | 파일 | 접근 가능 요소 |
|------|------|---------------|
| **register** | `register.js` | `this.element` (`appendElement`) |
| **beforeDestroy** | `beforeDestroy.js` | `this.element` |

### 페이지 라이프사이클 (참고)

| 단계 | 시점 | 역할 |
|------|------|------|
| **before_load** | 컴포넌트 register 이전 | 이벤트 핸들러 등록 |
| **loaded** | 컴포넌트 completed 이후 | 데이터 발행 및 interval 관리 |
| **before_unload** | 컴포넌트 beforeDestroy 이전 | 리소스 정리 |

---

## 작업 원칙

### 사용 전 필수 확인

**확실하지 않으면 추측하지 말고, 먼저 확인하거나 의논합니다.**

### 문제 해결 원칙

**문제가 발생하면 원인을 파악하는 것이 우선입니다.** 원인을 모르면서 이전 상태로 돌아가는 것은 해결이 아닙니다.

### 임시방편 금지 원칙

**`!important`나 임시방편은 근본 해결이 아닙니다.** 문제의 원인을 파악하고 구조적으로 해결해야 합니다.

### 스크린샷 검증 원칙

**스크린샷을 꼼꼼히 확인합니다.** 대충 보고 "정상"이라고 판단하거나, 사용자 반응에 맞춰 없는 문제를 지어내면 안 됩니다.

### 단계별 확인 원칙

**서두르지 말고 한 단계씩 확인합니다.** 변경 후 바로 다음 작업으로 넘어가지 않고, 결과를 확인한 뒤 진행합니다.

---

## fx.js 함수형 프로그래밍

컴포넌트 코드는 가능한 **fx.js**를 활용하여 함수형으로 작성합니다.

### 기본 함수

| 함수 | 용도 | 예시 |
|------|------|------|
| `fx.go` | 파이프라인 실행 | `fx.go(data, fx.filter(...), fx.map(...))` |
| `fx.pipe` | 파이프라인 함수 생성 | `const process = fx.pipe(filter, map)` |
| `fx.each` | 순회 (부수효과) | `fx.each(item => console.log(item), list)` |
| `fx.map` | 변환 | `fx.map(x => x * 2, [1,2,3])` |
| `fx.filter` | 필터링 | `fx.filter(x => x > 0, list)` |
| `fx.reduce` | 축약 | `fx.reduce((a, b) => a + b, 0, list)` |
| `fx.find` | 검색 | `fx.find(x => x.id === 1, list)` |
| `fx.take` | N개 추출 | `fx.take(5, list)` |

### 활용 패턴

#### 구독 등록 (fx.go + fx.each)
```javascript
fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => this[fn] && subscribe(topic, this, this[fn]), fnList)
    )
);
```

#### 필드 렌더링 (fx.go + fx.each)
```javascript
fx.go(
    config.fields,
    fx.each(({ key, selector, suffix }) => {
        const el = this.element.querySelector(selector);
        if (el) el.textContent = suffix ? `${data[key]}${suffix}` : data[key];
    })
);
```

#### 데이터 변환 (fx.go + fx.map + fx.filter)
```javascript
const activeItems = fx.go(
    data.items,
    fx.filter(item => item.status === 'active'),
    fx.map(item => ({ ...item, label: `[${item.id}] ${item.name}` })),
    fx.take(10)
);
```

#### 집계 (fx.go + fx.reduce)
```javascript
const total = fx.go(
    data.items,
    fx.map(item => item.value),
    fx.reduce((a, b) => a + b, 0)
);
```

#### 파이프라인 함수 재사용 (fx.pipe)
```javascript
const processItems = fx.pipe(
    fx.filter(item => item.value > 0),
    fx.map(item => ({ ...item, percent: item.value / 100 })),
    fx.take(10)
);

// 재사용
const cpuItems = processItems(cpuData);
const gpuItems = processItems(gpuData);
```

### 명령형 vs 함수형

```javascript
// ❌ 명령형
const results = [];
for (const item of data.items) {
    if (item.value > 50) {
        results.push({ ...item, highlight: true });
    }
}

// ✅ 함수형 (fx.js)
const results = fx.go(
    data.items,
    fx.filter(item => item.value > 50),
    fx.map(item => ({ ...item, highlight: true }))
);
```

---

## 핵심 패턴

### 1. PUB-SUB 패턴 (GlobalDataPublisher)

```javascript
const { subscribe, unsubscribe } = GlobalDataPublisher;

// ==================
// BINDINGS
// ==================

this.renderData = renderData.bind(this, config);

// ==================
// SUBSCRIPTIONS
// ==================

this.subscriptions = {
    topicA: ['renderData'],
    topicB: ['renderList', 'updateCount']
};

// 구독 등록
fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => this[fn] && subscribe(topic, this, this[fn]), fnList)
    )
);
```

### 2. Event-Driven 패턴 (WEventBus)

```javascript
const { bindEvents, removeCustomEvents } = WKit;

// ==================
// CUSTOM EVENTS
// ==================

this.customEvents = {
    click: {
        '.btn-refresh': '@refreshClicked',
        '.row-item': '@rowClicked'
    },
    change: {
        '.filter-select': '@filterChanged'
    }
};

bindEvents(this, this.customEvents);
```

### 3. Config 패턴 (What to render)

#### Field Config
```javascript
const config = {
    fields: [
        { key: 'name', selector: '.ups-name' },
        { key: 'status', selector: '.ups-status', dataAttr: 'status' },
        { key: 'load', selector: '.ups-load', suffix: '%' }
    ]
};
```

#### Chart Config (ECharts)
```javascript
const chartConfig = {
    xKey: 'timestamps',
    series: [
        { yKey: 'load', name: 'Load', color: '#3b82f6', smooth: true },
        { yKey: 'battery', name: 'Battery', color: '#22c55e' }
    ],
    optionBuilder: getMultiLineChartOption
};
```

#### Table Config (Tabulator)
```javascript
const tableConfig = {
    columns: [
        { title: 'ID', field: 'id', width: 60, hozAlign: 'center' },
        { title: 'Name', field: 'name', widthGrow: 2 },
        { title: 'Status', field: 'status', width: 100 }
    ],
    optionBuilder: getTableOption
};
```

#### Summary Config
```javascript
const summaryConfig = [
    { key: 'revenue', label: 'Revenue', icon: '💰', format: v => `$${v.toLocaleString()}` },
    { key: 'orders', label: 'Orders', icon: '📦', format: v => v.toLocaleString() }
];
```

### 4. TBD 패턴 (API 없이 미리 개발)

```javascript
// API 필드명이 미정일 때
const config = {
    titleKey: 'TBD_title',
    logsKey: 'TBD_logs'
};

this.subscriptions = {
    TBD_topicName: ['renderData']
};

this.customEvents = {
    click: {
        '.btn-clear': '@TBD_clearClicked'
    }
};
```

### 5. 응답 구조 패턴

```javascript
// 런타임 응답 구조: response 키가 한 번 더 감싸져 있음
// { response: { data, meta, ... } }

function renderData(config, { response }) {
    const { data, meta } = response;
    if (!data) return;
    // 렌더링 로직
}
```

### 6. 동적 리스트 렌더링 패턴 (Template Clone)

**사용 시점**: 리스트 아이템 개수가 데이터에 따라 변하는 경우

#### HTML (template 요소 사용)
```html
<div class="list-container">
    <div class="list">
        <!-- Template: 런타임에 복제될 아이템 구조 -->
        <template id="list-item-template">
            <div class="list__item">
                <span class="item__rank">1</span>
                <span class="item__name">-</span>
                <div class="item__progress">
                    <div class="progress__bar" style="--progress: 0%;"></div>
                </div>
                <span class="item__value">0%</span>
            </div>
        </template>
    </div>
</div>
```

#### register.js
```javascript
const config = {
    selectors: {
        list: '.list',
        template: '#list-item-template',
        // 아이템 내부 셀렉터 (아이템 기준 상대 경로)
        rank: '.item__rank',
        name: '.item__name',
        progressBar: '.progress__bar',
        value: '.item__value'
    },
    fields: {
        rank: 'TBD_rank',
        name: 'TBD_name',
        value: 'TBD_value'
    }
};

function renderList(config, { response }) {
    const { data } = response;
    if (!data || !data.items) return;

    const root = this.element;
    const list = root.querySelector(config.selectors.list);
    const template = root.querySelector(config.selectors.template);

    if (!list || !template) return;

    // 1. 기존 아이템 제거 (template 제외)
    list.querySelectorAll('.list__item').forEach(item => item.remove());

    // 2. 데이터 기반 아이템 생성
    data.items.forEach((itemData, index) => {
        // template 복제
        const clone = template.content.cloneNode(true);
        const item = clone.querySelector('.list__item');

        // 필드 바인딩
        const rankEl = item.querySelector(config.selectors.rank);
        if (rankEl) rankEl.textContent = itemData[config.fields.rank] ?? (index + 1);

        const nameEl = item.querySelector(config.selectors.name);
        if (nameEl) nameEl.textContent = itemData[config.fields.name] ?? '-';

        const progressBar = item.querySelector(config.selectors.progressBar);
        const value = itemData[config.fields.value] ?? 0;
        if (progressBar) progressBar.style.setProperty('--progress', `${value}%`);

        const valueEl = item.querySelector(config.selectors.value);
        if (valueEl) valueEl.textContent = `${value}%`;

        // 이벤트용 데이터 저장
        item.dataset.index = index;

        // 3. 리스트에 추가
        list.appendChild(item);
    });
}
```

#### 고정 개수 vs 동적 개수

| 유형 | 특징 | 패턴 |
|------|------|------|
| **고정 개수** | 아이템 수가 정해짐 (예: 2개 섹션) | `querySelectorAll` + 인덱스 매칭 |
| **동적 개수** | 아이템 수가 데이터에 따라 변함 | `template` + `cloneNode` |

```javascript
// 고정 개수: BusinessStatus 방식
const sections = root.querySelectorAll('.item[data-section]');
data.sections.forEach((sectionData, i) => {
    const sectionEl = sections[i];
    // 값 업데이트만
});

// 동적 개수: PerformanceMonitoring 방식
list.querySelectorAll('.list__item').forEach(item => item.remove());
data.items.forEach(itemData => {
    const clone = template.content.cloneNode(true);
    // 값 설정 후 appendChild
});
```

---

## register.js 템플릿

```javascript
/**
 * [ComponentName] Component - register.js
 *
 * 책임:
 * - [컴포넌트 역할 설명]
 *
 * Subscribes to: [topic명]
 * Events: [@이벤트명]
 */

const { subscribe } = GlobalDataPublisher;
const { bindEvents } = WKit;

// ==================
// CONFIG
// ==================

const config = {
    fields: [
        { key: 'fieldName', selector: '.field-selector' }
    ]
};

// ==================
// BINDINGS
// ==================

this.renderData = renderData.bind(this, config);

// ==================
// SUBSCRIPTIONS
// ==================

this.subscriptions = {
    topicName: ['renderData']
};

fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => this[fn] && subscribe(topic, this, this[fn]), fnList)
    )
);

// ==================
// CUSTOM EVENTS
// ==================

this.customEvents = {
    click: {
        '.btn-action': '@actionClicked'
    }
};

bindEvents(this, this.customEvents);

console.log('[ComponentName] Registered');

// ==================
// RENDER FUNCTIONS
// ==================

/**
 * 데이터 렌더링
 *
 * @param {Object} config - Field Config
 * @param {Object} param - API 응답 { response: { data } }
 */
function renderData(config, { response }) {
    const { data } = response;
    if (!data) return;

    fx.go(
        config.fields,
        fx.each(({ key, selector, suffix, dataAttr }) => {
            const el = this.element.querySelector(selector);
            if (!el) return;

            const value = data[key];
            if (dataAttr) {
                el.dataset[dataAttr] = value;
            } else {
                el.textContent = suffix ? `${value}${suffix}` : value;
            }
        })
    );

    console.log('[ComponentName] Rendered');
}
```

---

## beforeDestroy.js 템플릿

```javascript
/**
 * [ComponentName] Component - beforeDestroy.js
 */

const { unsubscribe } = GlobalDataPublisher;
const { removeCustomEvents } = WKit;

// ==================
// UNSUBSCRIBE
// ==================

if (this.subscriptions) {
    fx.go(
        Object.entries(this.subscriptions),
        fx.each(([topic, _]) => unsubscribe(topic, this))
    );
    this.subscriptions = null;
}

// ==================
// REMOVE EVENTS
// ==================

if (this.customEvents) {
    removeCustomEvents(this, this.customEvents);
    this.customEvents = null;
}

// ==================
// CLEAR REFERENCES
// ==================

this.renderData = null;

console.log('[ComponentName] Destroyed');
```

---

## preview.html 템플릿

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>[ComponentName] Preview</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
            background: #f1f5f9;
            padding: 2rem;
        }
        #component-container {
            width: 524px;   /* Figma 프레임 크기 */
            height: 350px;
            margin: 0 auto;
        }
    </style>
    <link rel="stylesheet" href="styles/component.css">
</head>
<body>
    <div id="component-container">
        <!-- views/component.html 내용 복사 -->
    </div>

    <!-- Mock Data -->
    <script>
        const MOCK_DATA = {
            topicName: {
                success: true,
                data: {
                    fieldName: 'Sample Value'
                }
            }
        };
    </script>

    <!-- Render Test -->
    <script>
        const container = document.getElementById('component-container');

        function renderData(data) {
            const { fieldName } = data;
            container.querySelector('.field-selector').textContent = fieldName;
            console.log('[Preview] Rendered:', data);
        }

        // 초기 렌더링
        renderData(MOCK_DATA.topicName.data);
    </script>
</body>
</html>
```

---

## 생성/정리 매칭 테이블

| 생성 (register) | 정리 (beforeDestroy) |
|-----------------|----------------------|
| `this.subscriptions = {...}` | `this.subscriptions = null` |
| `subscribe(topic, this, handler)` | `unsubscribe(topic, this)` |
| `this.customEvents = {...}` | `this.customEvents = null` |
| `bindEvents(this, customEvents)` | `removeCustomEvents(this, customEvents)` |
| `this.renderData = fn.bind(this)` | `this.renderData = null` |
| `this.chartInstance = echarts.init(...)` | `this.chartInstance.dispose()` |
| `this.tableInstance = new Tabulator(...)` | `this.tableInstance.destroy()` |
| `this.resizeObserver = new ResizeObserver(...)` | `this.resizeObserver.disconnect()` |

---

## CSS 레이아웃 원칙

### 레이아웃 컨테이너: flexbox 우선

**레이아웃 컨테이너**에는 `position: absolute` 대신 **flexbox**를 사용합니다.

```css
/* ❌ absolute 레이아웃 */
.component {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
}

/* ✅ flexbox 레이아웃 */
.component {
    display: flex;
    flex-direction: column;
    height: 100%;
}
.component__content {
    flex: 1;
    min-height: 0;  /* overflow 스크롤 작동에 필요 */
}
```

**absolute 허용 케이스**:
- 배경 레이어 (`z-index: 0`으로 분리)
- 오버레이, 팝업
- 아이콘 내부 장식 요소

---

## 라이브러리 활용

| 라이브러리 | 용도 | 초기화 | 정리 |
|------------|------|--------|------|
| **ECharts** | 차트 | `echarts.init(container)` | `.dispose()` |
| **Tabulator** | 테이블 | `new Tabulator(selector, options)` | `.destroy()` |
| **ResizeObserver** | 리사이즈 감지 | `new ResizeObserver(callback)` | `.disconnect()` |

### Tabulator height: 100% 사용 시 주의

Tabulator에서 `height: '100%'`가 작동하려면 부모 컨테이너가 **명시적 높이**를 가져야 합니다.

**문제**: `position: absolute` + `top/bottom`으로 계산된 높이는 자식의 퍼센트 계산에 전달되지 않음

**해결**: flexbox 레이아웃 사용
```css
/* 부모 컨테이너 */
.component {
    display: flex;
    flex-direction: column;
    height: 100%;
}

/* 테이블 컨테이너 */
.table-container {
    flex: 1;
    min-height: 0;  /* flexbox에서 overflow 스크롤 작동에 필요 */
}
```

`min-height: 0`은 flexbox 자식이 부모를 넘어서 overflow될 때 스크롤이 작동하도록 합니다.

---

## 금지 사항

```
❌ 페이지 구조에 컴포넌트 의존
- 컴포넌트는 topic만 알면 됨
- 페이지의 내부 구조 참조 금지

❌ 생성 후 정리 누락
- subscribe 후 unsubscribe 필수
- bindEvents 후 removeCustomEvents 필수
- 인스턴스 생성 후 dispose/destroy 필수

❌ 라이프사이클 순서 위반
- register에서만 초기화
- beforeDestroy에서만 정리

❌ 응답 구조 잘못 사용
- function(response) ❌
- function({ response }) ✅
```

---

## 완료 체크리스트

```
- [ ] 정적 HTML 구조 분석 완료
- [ ] views/component.html 생성 (data-bind 속성 포함)
- [ ] styles/component.css 생성 (#component-container 스코프)
- [ ] register.js 작성
    - [ ] subscriptions 정의
    - [ ] customEvents 정의
    - [ ] Config 정의
    - [ ] 렌더 함수 바인딩 (config 커링)
    - [ ] 응답 구조 ({ response }) 사용
- [ ] beforeDestroy.js 작성
    - [ ] unsubscribe 호출
    - [ ] removeCustomEvents 호출
    - [ ] 모든 참조 null 처리
- [ ] preview.html 작성
    - [ ] Mock 데이터 정의
    - [ ] 독립 렌더링 테스트
- [ ] 브라우저에서 preview.html 열어 확인
```

---

## 참고 문서

| 문서 | 참고 시점 | 내용 |
|------|----------|------|
| `discussions/2025-12-30_component_standalone.md` | API/Figma 없이 컴포넌트 개발 시 | 미리 완성 가능한 것 vs TBD 항목 구분 |
| `discussions/2025-12-31_config_pattern_catalog.md` | Config 구조 설계 시 | Field, Chart, Table 등 Config 패턴 카탈로그 |

## 참고 예제

| 예제 | 참고 시점 | 특징 |
|------|----------|------|
| `RNBT_architecture/Examples/example_tutorial/` | 처음 시작할 때 | 기본 구조, 교육용 대시보드 |
| `RNBT_architecture/Projects/ECO/` | 실제 프로젝트 패턴 확인 시 | 데이터센터 관리, 다양한 컴포넌트 |
| `RNBT_architecture/Projects/HANA_BANK_HIT_Dev/` | 동적 리스트 구현 시 | PerformanceMonitoring (template clone 패턴) |
