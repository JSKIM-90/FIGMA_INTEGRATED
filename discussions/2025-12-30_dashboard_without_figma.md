# Figma 없이 대시보드 만들기

## 배경

Figma 디자인이 없는 상황에서도 대시보드 프로토타입이 필요한 경우가 있습니다:

- 기술 검증 (PoC)
- 내부 데모
- API 설계 전 UI 프로토타이핑
- 컴포넌트 라이브러리 구축

## 핵심 원칙: 컴포넌트 우선 개발

```
┌─────────────────────────────────────────────────────────────┐
│                    Component-First 접근                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 1: 독립 컴포넌트 개발                                │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│   │ Chart   │  │ Table   │  │ Card    │  ← Mock 데이터      │
│   │Component│  │Component│  │Component│                     │
│   └─────────┘  └─────────┘  └─────────┘                    │
│        │            │            │                          │
│        └────────────┼────────────┘                          │
│                     ▼                                       │
│   Phase 2: 페이지 통합                                       │
│   ┌─────────────────────────────────────┐                  │
│   │              Page                    │                  │
│   │  ┌───────┐ ┌───────┐ ┌───────┐     │ ← API 연동        │
│   │  │ Chart │ │ Table │ │ Card  │     │                   │
│   │  └───────┘ └───────┘ └───────┘     │                   │
│   │         page_scripts/               │                   │
│   └─────────────────────────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**핵심**: 컴포넌트는 API나 페이지 스크립트 없이도 독립 동작 가능해야 함

---

## 워크플로우

### Phase 1: 독립 컴포넌트 개발

각 컴포넌트는 `preview.html`로 독립 테스트 가능

```
Projects/[프로젝트]/page/components/
├── SensorCard/
│   ├── views/component.html
│   ├── styles/component.css
│   ├── scripts/
│   │   ├── register.js
│   │   └── beforeDestroy.js
│   └── preview.html          ← 브라우저에서 직접 열기
│
├── DataTable/
│   └── preview.html
│
└── TrendChart/
    └── preview.html
```

#### preview.html 구조

```html
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="styles/component.css">
    <!-- 필요한 라이브러리 -->
    <script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
</head>
<body>
    <!-- 컴포넌트 마크업 -->
    <div id="component-root">
        <!-- views/component.html 내용 -->
    </div>

    <script>
        // Mock 컨텍스트
        const mockContext = {
            element: document.getElementById('component-root'),
            setter: {
                // 컴포넌트별 setter 데이터
            }
        };

        // Mock 데이터
        const mockData = {
            temperature: 24.5,
            humidity: 65,
            status: 'normal'
        };

        // register.js 로직 실행
        // ...
    </script>
</body>
</html>
```

### Phase 2: 페이지 통합

컴포넌트가 준비되면 페이지에서 조합

```
Projects/[프로젝트]/
├── page/
│   ├── components/           # Phase 1에서 개발한 컴포넌트들
│   │   ├── SensorCard/
│   │   ├── DataTable/
│   │   └── TrendChart/
│   │
│   ├── page_scripts/         # Phase 2에서 추가
│   │   ├── before_load.js    # 이벤트 핸들러 등록
│   │   ├── loaded.js         # 데이터 fetch + publish
│   │   └── before_unload.js  # 정리
│   │
│   └── views/
│       └── page.html         # 컴포넌트 배치
│
└── mock_server/              # Phase 2에서 추가
    ├── server.js
    └── routes/
```

---

## Config 패턴: API 미정 상태 대응

API가 결정되지 않은 상태에서 config 패턴을 도입하면, 나중에 API가 확정되었을 때 **코드 수정 없이 config만 변경**하면 됩니다.

### 데이터 바인딩 Config

```javascript
// register.js
this.dataBindConfig = [
    { key: 'temperature', selector: '.temp', suffix: '°C' },
    { key: 'humidity', selector: '.humidity', suffix: '%' },
    { key: 'status', selector: '.status', dataAttr: 'status' }
];

this.renderData = function(data) {
    this.dataBindConfig.forEach(({ key, selector, suffix, dataAttr }) => {
        const el = this.element.querySelector(selector);
        if (el) {
            el.textContent = data[key] + (suffix || '');
            if (dataAttr) el.dataset[dataAttr] = data[key];
        }
    });
};
```

API 응답 필드명이 바뀌면 config만 수정:

```javascript
// 변경 전: { temperature: 24.5 }
// 변경 후: { temp_value: 24.5 }

this.dataBindConfig = [
    { key: 'temp_value', selector: '.temp', suffix: '°C' },  // key만 변경
    // ...
];
```

### 차트 Config

```javascript
this.chartConfig = {
    xKey: 'timestamps',           // API 응답의 x축 필드명
    series: [
        { yKey: 'temperatures', name: '온도', color: '#3b82f6' },
        { yKey: 'humidities', name: '습도', color: '#22c55e' }
    ]
};

this.renderChart = function(data) {
    const option = buildChartOption(this.chartConfig, data);
    this.chart.setOption(option);
};
```

API 필드명이 `temperatures` → `temp_history`로 바뀌면:

```javascript
this.chartConfig = {
    xKey: 'time_labels',
    series: [
        { yKey: 'temp_history', name: '온도', color: '#3b82f6' },  // yKey만 변경
        // ...
    ]
};
```

### 데이터셋 Config

```javascript
this.datasetInfo = [
    { datasetName: 'sensor', param: { id: assetId }, render: ['renderData'] },
    { datasetName: 'sensorHistory', param: { id: assetId }, render: ['renderChart'] }
];
```

API 엔드포인트가 바뀌면 `datasetName`만 수정 (Mock Server의 routes에서 매핑).

### Config 패턴의 장점

| 변경 사항 | 코드 수정 | Config 수정 |
|-----------|----------|-------------|
| API 필드명 변경 | ✗ | ✓ (key만 수정) |
| 표시 형식 변경 | ✗ | ✓ (suffix, format 수정) |
| 차트 시리즈 추가 | ✗ | ✓ (series 배열에 추가) |
| 색상 변경 | ✗ | ✓ (color 수정) |

---

## 컴포넌트 단계에서 register.js / beforeDestroy.js 작성

API가 미정이어도 컴포넌트의 스크립트는 미리 완성할 수 있습니다.

### register.js 템플릿

```javascript
/**
 * SensorCard - register.js
 * API 미정 상태에서도 완성 가능한 구조
 */

const { GlobalDataPublisher, WEventBus } = WKit;

// ======================
// CONFIG (API 확정 시 수정)
// ======================
this.dataBindConfig = [
    { key: 'temperature', selector: '.temp', suffix: '°C' },
    { key: 'humidity', selector: '.humidity', suffix: '%' },
    { key: 'status', selector: '.status', dataAttr: 'status' }
];

this.datasetInfo = [
    { datasetName: 'sensorData', param: {}, render: ['renderData'] }
];

// ======================
// CUSTOM EVENTS (완성)
// ======================
this.customEvents = {
    click: '@sensorCardClicked',
    dblclick: '@sensorCardDblClicked'
};

// ======================
// RENDER METHODS (완성)
// ======================
this.renderData = function(data) {
    this.dataBindConfig.forEach(({ key, selector, suffix, dataAttr }) => {
        const el = this.element.querySelector(selector);
        if (el) {
            el.textContent = data[key] + (suffix || '');
            if (dataAttr) el.dataset[dataAttr] = data[key];
        }
    });
};

// ======================
// EVENT BINDINGS (완성)
// ======================
Object.entries(this.customEvents).forEach(([eventType, eventName]) => {
    this.element.addEventListener(eventType, (e) => {
        WEventBus.emit(eventName, { targetInstance: this, event: e });
    });
});

// ======================
// SUBSCRIPTION (구독 패턴)
// ======================
this.subscriptions = [];

this.subscriptions.push(
    GlobalDataPublisher.subscribe('sensorData', (response) => {
        this.renderData(response.data);
    })
);

console.log('[SensorCard] Registered:', this.id);
```

### beforeDestroy.js 템플릿

```javascript
/**
 * SensorCard - beforeDestroy.js
 * 구독 해제 + 이벤트 정리
 */

// 구독 해제 (배열 패턴)
if (this.subscriptions) {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions = [];
}

console.log('[SensorCard] Destroyed:', this.id);
```

### 차트 컴포넌트 예시

```javascript
// TrendChart/scripts/register.js

const { GlobalDataPublisher } = WKit;

// ======================
// CONFIG
// ======================
this.chartConfig = {
    xKey: 'timestamps',
    series: [
        { yKey: 'values', name: 'Value', color: '#3b82f6' }
    ]
};

// ======================
// CHART SETUP (완성)
// ======================
this.chart = echarts.init(this.element.querySelector('.chart-container'));

this.renderChart = function(data) {
    const { xKey, series } = this.chartConfig;
    const option = {
        xAxis: { type: 'category', data: data[xKey] },
        yAxis: { type: 'value' },
        series: series.map(s => ({
            name: s.name,
            type: 'line',
            data: data[s.yKey],
            lineStyle: { color: s.color }
        }))
    };
    this.chart.setOption(option);
};

// ======================
// SUBSCRIPTION (구독 패턴)
// ======================
this.subscriptions = [];

this.subscriptions.push(
    GlobalDataPublisher.subscribe('chartData', (response) => {
        this.renderChart(response.data);
    })
);

console.log('[TrendChart] Registered:', this.id);
```

```javascript
// TrendChart/scripts/beforeDestroy.js

// 차트 정리
if (this.chart) {
    this.chart.dispose();
    this.chart = null;
}

// 구독 해제
if (this.subscriptions) {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions = [];
}

console.log('[TrendChart] Destroyed:', this.id);
```

### 컴포넌트 완성도

| 항목 | API 미정 | API 확정 후 |
|------|---------|------------|
| register.js | ✓ 완성 | config만 수정 |
| beforeDestroy.js | ✓ 완성 | 변경 없음 |
| preview.html | ✓ Mock으로 테스트 | 변경 없음 |
| customEvents | ✓ 완성 | 변경 없음 |
| subscriptions | ✓ 완성 (topic명만 수정) | topic명 확정 |

### 구독 패턴 규칙

```javascript
// ✓ DO: 배열로 관리
this.subscriptions = [];
this.subscriptions.push(GlobalDataPublisher.subscribe('topic1', cb1));
this.subscriptions.push(GlobalDataPublisher.subscribe('topic2', cb2));

// ✓ DO: 일괄 해제
this.subscriptions.forEach(sub => sub.unsubscribe());

// ✗ DON'T: 개별 변수로 관리
this.sub1 = GlobalDataPublisher.subscribe('topic1', cb1);
this.sub2 = GlobalDataPublisher.subscribe('topic2', cb2);
```

### 이벤트 정의 규칙

```javascript
// ✓ DO: customEvents 객체로 정의
this.customEvents = {
    click: '@componentClicked',
    dblclick: '@componentDblClicked'
};

// ✓ DO: 일괄 바인딩
Object.entries(this.customEvents).forEach(([eventType, eventName]) => {
    this.element.addEventListener(eventType, (e) => {
        WEventBus.emit(eventName, { targetInstance: this, event: e });
    });
});
```

---

## 실제 예시: 센서 모니터링 대시보드

### Step 1: 개별 컴포넌트 개발 (API 없이)

**SensorCard** - 온습도 표시
```javascript
// register.js
this.renderData = function(data) {
    this.element.querySelector('.temp').textContent = data.temperature + '°C';
    this.element.querySelector('.humidity').textContent = data.humidity + '%';
};

// preview.html에서 Mock 데이터로 테스트
this.renderData({ temperature: 24.5, humidity: 65 });
```

**TrendChart** - 히스토리 차트
```javascript
// register.js
this.chart = echarts.init(this.element.querySelector('.chart'));

this.renderChart = function(data) {
    this.chart.setOption({
        xAxis: { data: data.timestamps },
        series: [{ data: data.values }]
    });
};

// preview.html에서 Mock 데이터로 테스트
this.renderChart({
    timestamps: ['10:00', '10:05', '10:10'],
    values: [24, 25, 24.5]
});
```

### Step 2: 구독 패턴 추가

API가 결정되면 구독 패턴 추가

```javascript
// register.js 수정
const { GlobalDataPublisher } = WKit;

this.subscription = GlobalDataPublisher.subscribe('sensorData', (response) => {
    this.renderData(response.data);
});

// renderData 메서드는 그대로 유지
```

### Step 3: 페이지 스크립트 작성

```javascript
// page_scripts/loaded.js
const { fetchAndPublish } = GlobalDataPublisher;

fetchAndPublish('sensorData', this);  // API 호출 + 발행
```

---

## 장점

### 1. 병렬 개발 가능

```
개발자 A: SensorCard 컴포넌트 개발
개발자 B: TrendChart 컴포넌트 개발
개발자 C: API 설계 + Mock Server
         ↓
      페이지 통합
```

### 2. 테스트 용이

- 각 컴포넌트를 `preview.html`로 독립 테스트
- API 의존 없이 UI 검증 가능

### 3. 점진적 발전

```
Phase 1: Mock 데이터로 UI 완성
Phase 2: 구독 패턴 추가
Phase 3: 실제 API 연동
Phase 4: Figma 디자인 적용 (선택)
```

---

## 체크리스트

### 컴포넌트 개발 시

- [ ] `preview.html`로 독립 실행 가능한가?
- [ ] Mock 데이터로 모든 상태 테스트 가능한가?
- [ ] `renderXxx` 메서드가 데이터만 받으면 동작하는가?

### 페이지 통합 시

- [ ] 컴포넌트가 구독 패턴을 사용하는가?
- [ ] `page_scripts/loaded.js`에서 `fetchAndPublish` 호출하는가?
- [ ] `beforeDestroy.js`에서 구독 해제하는가?

---

## 결론

Figma 디자인이 없어도:
1. **컴포넌트 우선 개발**로 UI 로직 구현
2. **preview.html**로 독립 테스트
3. API와 페이지 스크립트는 **나중에 결정** 가능

핵심은 **컴포넌트의 독립성**입니다. 데이터를 받으면 렌더링하는 순수한 컴포넌트를 먼저 만들고, 데이터가 어디서 오는지(API, WebSocket, Mock)는 나중에 결정합니다.

---

*작성일: 2025-12-30*
