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
├── views/component.html       # 데이터 바인딩 마크업
├── styles/component.css       # 스타일 (#component-container 스코프)
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

## 라이브러리 활용

| 라이브러리 | 용도 | 초기화 | 정리 |
|------------|------|--------|------|
| **ECharts** | 차트 | `echarts.init(container)` | `.dispose()` |
| **Tabulator** | 테이블 | `new Tabulator(selector, options)` | `.destroy()` |
| **ResizeObserver** | 리사이즈 감지 | `new ResizeObserver(callback)` | `.disconnect()` |

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

## 참고 예제

- `RNBT_architecture/Examples/example_tutorial/` - 교육용 대시보드
- `RNBT_architecture/Projects/ECO/` - 데이터센터 관리
- `RNBT_architecture/discussions/2025-12-31_config_pattern_catalog.md` - Config 패턴 카탈로그
