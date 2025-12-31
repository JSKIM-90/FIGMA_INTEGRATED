---
name: create-dashboard
description: RNBT 아키텍처 패턴에 맞는 완전한 대시보드 페이지를 생성합니다. Master/Page 레이어, 여러 컴포넌트, Mock 서버, datasetList.json을 포함한 전체 구조를 생성합니다. Use when creating dashboard pages, implementing Master/Page architecture, or setting up complete page structures with multiple components.
---

# RNBT 대시보드 생성

RNBT 아키텍처 패턴에 맞는 **완전한 대시보드 페이지**를 생성하는 Skill입니다.
Master/Page 레이어, 컴포넌트들, Mock 서버, datasetList.json을 포함합니다.

---

## 출력 구조

```
RNBT_architecture/Examples/[example_name]/
├── mock_server/                    # Express API 서버
│   ├── server.js
│   └── package.json
│
├── master/                         # MASTER 레이어 (앱 전역)
│   └── page/
│       ├── page_scripts/
│       │   ├── before_load.js      # 이벤트 핸들러 등록
│       │   ├── loaded.js           # 데이터 매핑 및 발행
│       │   └── before_unload.js    # 리소스 정리
│       └── components/
│           ├── Header/
│           └── Sidebar/
│
├── page/                           # PAGE 레이어 (페이지별)
│   ├── page_scripts/
│   │   ├── before_load.js          # 이벤트 핸들러 + currentParams
│   │   ├── loaded.js               # 데이터 매핑 + interval
│   │   └── before_unload.js        # 리소스 정리
│   └── components/
│       ├── StatsCards/
│       ├── DataTable/
│       └── TrendChart/
│
├── datasetList.json                # API 엔드포인트 정의
└── README.md
```

---

## Master vs Page 레이어

| 레이어 | 범위 | 용도 | 예시 컴포넌트 |
|--------|------|------|--------------|
| **Master** | 앱 전역 | 공통 UI, 사용자 정보, 네비게이션 | Header, Sidebar |
| **Page** | 페이지별 | 페이지 고유 컴포넌트, 데이터 | StatsCards, DataTable, TrendChart |

---

## 라이프사이클 흐름

```
앱 시작
  ↓
[MASTER] before_load.js
  - eventBusHandlers 등록 (네비게이션 등)
  ↓
[MASTER] 컴포넌트 register.js
  - Header, Sidebar 초기화
  ↓
[MASTER] loaded.js
  - userInfo, menuList 발행
  ↓
페이지 진입
  ↓
[PAGE] before_load.js
  - eventBusHandlers 등록
  - currentParams 초기화
  ↓
[PAGE] 컴포넌트 register.js
  - StatsCards, DataTable, TrendChart 초기화
  ↓
[PAGE] loaded.js
  - globalDataMappings 등록
  - fetchAndPublish
  - startAllIntervals
  ↓
페이지 이탈
  ↓
[PAGE] before_unload.js
  - stopAllIntervals
  - offEventBusHandlers
  - unregisterMapping
  ↓
[PAGE] 컴포넌트 beforeDestroy.js
  ↓
앱 종료
  ↓
[MASTER] before_unload.js
  ↓
[MASTER] 컴포넌트 beforeDestroy.js
```

---

## 파일 템플릿

### datasetList.json

```json
{
  "version": "3.2.0",
  "data": [
    {
      "datasource": "",
      "mode": "0",
      "delivery_type": "0",
      "param_info": [],
      "data_type": "1",
      "interval": "",
      "page_id": "MASTER",
      "dataset_id": "user-001",
      "name": "userApi",
      "rest_api": "{\"url\":\"http://localhost:3003/api/user\",\"method\":\"GET\",\"headers\":{},\"body\":\"\"}"
    },
    {
      "datasource": "",
      "mode": "0",
      "delivery_type": "0",
      "param_info": [
        {"param_name": "category", "param_type": "string", "default_value": "all"}
      ],
      "data_type": "1",
      "interval": "30000",
      "page_id": "PAGE",
      "dataset_id": "table-001",
      "name": "tableApi",
      "rest_api": "{\"url\":\"http://localhost:3003/api/data?category=#{category}\",\"method\":\"GET\",\"headers\":{},\"body\":\"\"}"
    }
  ],
  "datasource": []
}
```

**주의사항:**
- `rest_api`는 JSON 문자열로 이스케이프
- `param_info`는 배열 형태
- `interval`은 밀리초 문자열 (예: "30000")
- `page_id`는 "MASTER" 또는 "PAGE"

### page/page_scripts/before_load.js

```javascript
/**
 * PAGE - before_load.js
 *
 * 시점: 컴포넌트 register 이전
 * 책임: 이벤트 핸들러 등록, currentParams 초기화
 */

const { onEventBusHandlers } = WEventBus;

// ==================
// CURRENT PARAMS
// ==================

this.currentParams = {
    tableData: { category: 'all' },
    chartData: { period: '7d' }
};

// ==================
// EVENT BUS HANDLERS
// ==================

this.eventBusHandlers = {
    '@filterChanged': ({ event }) => {
        const category = event.target.value;
        this.currentParams.tableData = { category };
        GlobalDataPublisher.fetchAndPublish('tableData', this, this.currentParams.tableData);
        console.log('[Page] Filter changed:', category);
    },

    '@periodChanged': ({ event }) => {
        const period = event.target.value;
        this.currentParams.chartData = { period };
        GlobalDataPublisher.fetchAndPublish('chartData', this, this.currentParams.chartData);
        console.log('[Page] Period changed:', period);
    },

    '@rowClicked': ({ data }) => {
        console.log('[Page] Row clicked:', data);
    },

    '@cardClicked': ({ event }) => {
        const key = event.currentTarget.dataset.statKey;
        console.log('[Page] Card clicked:', key);
    }
};

onEventBusHandlers(this.eventBusHandlers);

console.log('[Page] before_load completed');
```

### page/page_scripts/loaded.js

```javascript
/**
 * PAGE - loaded.js
 *
 * 시점: 컴포넌트 completed 이후
 * 책임: 데이터 매핑 등록, 초기 발행, interval 시작
 */

const { registerMapping, fetchAndPublish, startAllIntervals } = GlobalDataPublisher;

// ==================
// DATA MAPPINGS
// ==================

this.globalDataMappings = [
    {
        topic: 'stats',
        datasetName: 'statsApi',
        param: {}
    },
    {
        topic: 'tableData',
        datasetName: 'tableApi',
        param: this.currentParams.tableData
    },
    {
        topic: 'chartData',
        datasetName: 'chartApi',
        param: this.currentParams.chartData
    }
];

// 매핑 등록
fx.go(
    this.globalDataMappings,
    fx.each(mapping => registerMapping(this, mapping))
);

// ==================
// REFRESH INTERVALS
// ==================

this.refreshIntervals = {
    stats: 10000,      // 10초
    tableData: 30000,  // 30초
    chartData: 15000   // 15초
};

// ==================
// INITIAL FETCH
// ==================

fx.go(
    this.globalDataMappings,
    fx.each(({ topic }) => {
        const param = this.currentParams[topic] || {};
        fetchAndPublish(topic, this, param);
    })
);

// ==================
// START INTERVALS
// ==================

startAllIntervals(this, this.refreshIntervals, this.currentParams);

console.log('[Page] loaded completed');
```

### page/page_scripts/before_unload.js

```javascript
/**
 * PAGE - before_unload.js
 *
 * 시점: 컴포넌트 beforeDestroy 이전
 * 책임: interval 정지, 이벤트 해제, 매핑 해제
 */

const { offEventBusHandlers } = WEventBus;
const { stopAllIntervals, unregisterMapping } = GlobalDataPublisher;

// ==================
// STOP INTERVALS
// ==================

if (this.refreshIntervals) {
    stopAllIntervals(this);
    this.refreshIntervals = null;
}

// ==================
// OFF EVENT HANDLERS
// ==================

if (this.eventBusHandlers) {
    offEventBusHandlers(this.eventBusHandlers);
    this.eventBusHandlers = null;
}

// ==================
// UNREGISTER MAPPINGS
// ==================

if (this.globalDataMappings) {
    fx.go(
        this.globalDataMappings,
        fx.each(mapping => unregisterMapping(this, mapping))
    );
    this.globalDataMappings = null;
}

// ==================
// CLEAR PARAMS
// ==================

this.currentParams = null;

console.log('[Page] before_unload completed');
```

---

## 컴포넌트 유형별 구현

### StatsCards (Summary Config 패턴)

```javascript
// register.js
const summaryConfig = [
    { key: 'revenue', label: 'Revenue', icon: '💰', format: (v, unit) => `${unit}${v.toLocaleString()}` },
    { key: 'orders', label: 'Orders', icon: '📦', format: v => v.toLocaleString() },
    { key: 'customers', label: 'Customers', icon: '👥', format: v => v.toLocaleString() },
    { key: 'conversion', label: 'Conversion', icon: '📈', format: (v, unit) => `${v}${unit}` }
];

this.subscriptions = { stats: ['renderStats'] };
this.customEvents = { click: { '.stat-card': '@cardClicked' } };
```

### DataTable (Table Config + Tabulator)

```javascript
// register.js
const tableConfig = {
    columns: [
        { title: 'ID', field: 'id', width: 60, hozAlign: 'center' },
        { title: 'Product', field: 'product', widthGrow: 2 },
        { title: 'Category', field: 'category', width: 120 },
        { title: 'Price', field: 'price', width: 100, hozAlign: 'right',
          formatter: cell => `$${cell.getValue().toLocaleString()}` }
    ]
};

this.tableInstance = new Tabulator(`#${uniqueId}`, {
    layout: 'fitColumns',
    height: '100%',
    placeholder: 'No data available',
    columns: tableConfig.columns
});

this.tableInstance.on('tableBuilt', () => {
    // 데이터 로드
});

this.subscriptions = { tableData: ['renderTable'] };
this.customEvents = { change: { '.filter-select': '@filterChanged' } };
```

### TrendChart (Chart Config + ECharts)

```javascript
// register.js
const chartConfig = {
    xKey: 'labels',
    seriesKey: 'series',
    optionBuilder: getChartOptions
};

this.chartInstance = echarts.init(chartContainer);

this.resizeObserver = new ResizeObserver(() => {
    this.chartInstance && this.chartInstance.resize();
});
this.resizeObserver.observe(chartContainer);

this.subscriptions = { chartData: ['renderChart'] };
this.customEvents = { change: { '.period-select': '@periodChanged' } };
```

---

## Mock Server 템플릿

### mock_server/package.json

```json
{
  "name": "mock-server",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.18.2"
  }
}
```

### mock_server/server.js

```javascript
const express = require('express');
const cors = require('cors');
const app = express();
const PORT = 3003;

app.use(cors());
app.use(express.json());

// ==================
// MASTER ENDPOINTS
// ==================

app.get('/api/user', (req, res) => {
    res.json({
        success: true,
        data: {
            name: 'John Doe',
            role: 'Administrator',
            avatar: 'https://via.placeholder.com/32'
        }
    });
});

app.get('/api/menu', (req, res) => {
    res.json({
        success: true,
        items: [
            { id: 'dashboard', label: 'Dashboard', icon: 'home', active: true },
            { id: 'analytics', label: 'Analytics', icon: 'chart' },
            { id: 'settings', label: 'Settings', icon: 'gear' }
        ]
    });
});

// ==================
// PAGE ENDPOINTS
// ==================

app.get('/api/stats', (req, res) => {
    res.json({
        success: true,
        data: {
            revenue: { value: 125000, unit: '$', change: 12.5 },
            orders: { value: 1234, unit: '', change: 8.2 },
            customers: { value: 567, unit: '', change: -2.1 },
            conversion: { value: 3.2, unit: '%', change: 0.5 }
        }
    });
});

app.get('/api/data', (req, res) => {
    const { category } = req.query;
    let data = [
        { id: 1, product: 'Product A', category: 'electronics', price: 299 },
        { id: 2, product: 'Product B', category: 'clothing', price: 59 },
        { id: 3, product: 'Product C', category: 'electronics', price: 199 }
    ];

    if (category && category !== 'all') {
        data = data.filter(item => item.category === category);
    }

    res.json({
        success: true,
        data,
        meta: { total: data.length, category }
    });
});

app.get('/api/trend', (req, res) => {
    const { period } = req.query;
    const labels = period === '24h'
        ? ['00:00', '04:00', '08:00', '12:00', '16:00', '20:00']
        : period === '7d'
        ? ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
        : ['Week 1', 'Week 2', 'Week 3', 'Week 4'];

    res.json({
        success: true,
        data: {
            labels,
            series: [
                { name: 'Revenue', data: labels.map(() => Math.floor(Math.random() * 10000)), color: '#3b82f6' },
                { name: 'Orders', data: labels.map(() => Math.floor(Math.random() * 100)), color: '#22c55e' }
            ]
        },
        meta: { period }
    });
});

// ==================
// START SERVER
// ==================

app.listen(PORT, () => {
    console.log(`Mock server running at http://localhost:${PORT}`);
});
```

---

## 응답 구조 규칙

```javascript
// 런타임이 전달하는 응답 구조
// { response: { data, meta, ... } }

// 렌더 함수에서 destructuring
function renderData(config, { response }) {
    const { data, meta } = response;
    if (!data) return;
    // ...
}
```

---

## 생성/정리 매칭 테이블

### 페이지

| 생성 (before_load / loaded) | 정리 (before_unload) |
|-----------------------------|----------------------|
| `this.eventBusHandlers = {...}` | `this.eventBusHandlers = null` |
| `onEventBusHandlers(...)` | `offEventBusHandlers(...)` |
| `this.globalDataMappings = [...]` | `this.globalDataMappings = null` |
| `this.currentParams = {...}` | `this.currentParams = null` |
| `this.refreshIntervals = {...}` | `this.refreshIntervals = null` |
| `registerMapping(...)` | `unregisterMapping(...)` |
| `startAllIntervals(...)` | `stopAllIntervals(...)` |

### 컴포넌트

| 생성 (register) | 정리 (beforeDestroy) |
|-----------------|----------------------|
| `this.subscriptions = {...}` | `this.subscriptions = null` |
| `subscribe(topic, this, handler)` | `unsubscribe(topic, this)` |
| `this.customEvents = {...}` | `this.customEvents = null` |
| `bindEvents(this, customEvents)` | `removeCustomEvents(this, customEvents)` |
| `echarts.init(...)` | `.dispose()` |
| `new Tabulator(...)` | `.destroy()` |
| `new ResizeObserver(...)` | `.disconnect()` |

---

## 금지 사항

```
❌ datasetList.json 형식 임의 변경
- rest_api는 JSON 문자열로 이스케이프
- param_info는 배열 형태
- 기존 예제 형식 준수

❌ 생성/정리 불일치
- 모든 생성 리소스는 정리되어야 함
- interval, subscription, event 모두 해제

❌ 라이프사이클 순서 위반
- before_load: 이벤트 등록만
- loaded: 데이터 발행, interval 시작
- before_unload: 정리만

❌ 응답 구조 잘못 사용
- function(response) ❌
- function({ response }) ✅
```

---

## 완료 체크리스트

```
Master 레이어:
- [ ] master/page/page_scripts/before_load.js
- [ ] master/page/page_scripts/loaded.js
- [ ] master/page/page_scripts/before_unload.js
- [ ] master/page/components/Header/ (전체 구조)
- [ ] master/page/components/Sidebar/ (전체 구조)

Page 레이어:
- [ ] page/page_scripts/before_load.js
- [ ] page/page_scripts/loaded.js
- [ ] page/page_scripts/before_unload.js
- [ ] page/components/[각 컴포넌트]/ (전체 구조)

데이터:
- [ ] datasetList.json (기존 형식 준수)
- [ ] mock_server/server.js
- [ ] mock_server/package.json

문서:
- [ ] README.md

검증:
- [ ] mock_server 실행 (npm start)
- [ ] API 테스트 (curl)
- [ ] 각 컴포넌트 preview.html 확인
```

---

## 참고 예제

- `RNBT_architecture/Examples/example_tutorial/` - 교육용 대시보드 (이 Skill의 기반)
- `RNBT_architecture/Examples/example_basic_01/` - IoT 대시보드
- `RNBT_architecture/Examples/example_master_01/` - Master-Page 아키텍처
- `RNBT_architecture/Projects/ECO/` - 실전 데이터센터 관리
