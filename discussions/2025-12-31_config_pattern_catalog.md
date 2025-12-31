# Config 패턴 카탈로그

> 작성일: 2025-12-31
> 목적: RNBT_architecture에서 사용되는 Config 패턴의 체계적 정리

---

## 개요

Config 패턴은 **데이터 매핑과 렌더링 로직을 분리**하여 재사용성을 높이는 핵심 설계 원칙입니다.

```
메소드는 그대로, Config만 변경 → 다양한 데이터에 동일 로직 적용
```

### 핵심 원리

```javascript
// 1. Config 정의 (What to render)
const config = {
    key: 'temperature',
    selector: '.temp-value',
    suffix: '°C'
};

// 2. 렌더 함수 바인딩 (How to render)
this.renderData = renderData.bind(this, config);

// 3. 호출 시 Config가 첫 번째 인자로 전달
function renderData(config, response) {
    const { key, selector, suffix } = config;
    // config를 사용한 렌더링 로직
}
```

---

## Config 패턴 유형

### 1. Field Config (필드 매핑)

**용도**: API 응답 필드를 DOM 요소에 매핑

```javascript
// 기본 형태
this.baseInfoConfig = [
    { key: 'name', selector: '.ups-name' },
    { key: 'zone', selector: '.ups-zone' },
    { key: 'status', selector: '.ups-status', dataAttr: 'status' }
];

// 확장 형태 (suffix 포함)
this.upsInfoConfig = [
    { key: 'load', selector: '.ups-load', suffix: '%' },
    { key: 'batteryLevel', selector: '.ups-battery', suffix: '%' },
    { key: 'inputVoltage', selector: '.ups-input-voltage', suffix: 'V' },
    { key: 'outputVoltage', selector: '.ups-output-voltage', suffix: 'V' },
    { key: 'runtime', selector: '.ups-runtime', suffix: 'min' },
    { key: 'mode', selector: '.ups-mode' }
];
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `key` | string | O | API 응답 객체의 필드명 |
| `selector` | string | O | DOM 요소 CSS 선택자 |
| `suffix` | string | X | 값 뒤에 붙일 단위 (%, V, min 등) |
| `dataAttr` | string | X | data-* 속성으로 저장할 경우 속성명 |

**렌더 함수 패턴**:
```javascript
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
```

**사용 예시** (ECO/UPS):
```javascript
// Config 조합
this.renderUPSInfo = renderUPSInfo.bind(this, [...this.baseInfoConfig, ...this.upsInfoConfig]);
```

---

### 2. Chart Config (ECharts 설정)

**용도**: ECharts 옵션을 선언적으로 정의

#### 2.1 Line/Area Chart Config

```javascript
// TimeTrendChart - 다중 Area 차트
const config = {
    // X축 필드 매핑
    xKey: 'tm',

    // 시리즈 정의
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
    areaStyle: true,
    areaGradient: true,

    // Y축 설정
    yAxis: {
        min: 0,
        max: 1800,
        interval: 600
    }
};
```

#### 2.2 Multi-Line Chart Config (UPS/CRAC)

```javascript
this.chartConfig = {
    xKey: 'timestamps',
    series: [
        { yKey: 'load', name: 'Load', color: '#3b82f6', smooth: true, areaStyle: true },
        { yKey: 'battery', name: 'Battery', color: '#22c55e', smooth: true }
    ],
    optionBuilder: getMultiLineChartOption  // 옵션 빌더 함수 참조
};
```

#### 2.3 Dual Axis Chart Config (온습도 센서)

```javascript
this.chartConfig = {
    xKey: 'timestamps',
    series: [
        { yKey: 'temperatures', name: 'Temperature', color: '#3b82f6', yAxisIndex: 0 },
        { yKey: 'humidity', name: 'Humidity', color: '#22c55e', yAxisIndex: 1 }
    ],
    yAxis: [
        { name: '°C', position: 'left' },
        { name: '%', position: 'right' }
    ],
    optionBuilder: getDualAxisChartOption
};
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `xKey` | string | O | X축 데이터 필드명 |
| `series` / `seriesMap` | array | O | 시리즈 배열 |
| `series[].key` / `yKey` | string | O | Y축 데이터 필드명 |
| `series[].name` | string | O | 범례 표시명 |
| `series[].color` | string | O | 시리즈 색상 (hex) |
| `series[].smooth` | boolean | X | 곡선 여부 (default: false) |
| `series[].areaStyle` | boolean | X | Area 채움 여부 |
| `series[].yAxisIndex` | number | X | 이중 Y축 시 인덱스 (0, 1) |
| `yAxis` | object/array | X | Y축 설정 (단일 또는 이중) |
| `optionBuilder` | function | X | ECharts 옵션 생성 함수 |

**렌더 함수 패턴**:
```javascript
function renderChart(config, data) {
    const { optionBuilder, ...chartConfig } = config;
    const option = optionBuilder(chartConfig, data);
    this.updateChart('.chart-container', option);
}
```

---

### 3. Table Config (Tabulator 설정)

**용도**: Tabulator 테이블 컬럼 및 옵션 정의

```javascript
this.tableConfig = {
    selector: '.table-container',
    columns: [
        { title: 'ID', field: 'id', widthGrow: 0.5, hozAlign: 'right' },
        { title: 'Name', field: 'name', widthGrow: 1.5 },
        { title: 'Load', field: 'load', widthGrow: 1, hozAlign: 'right',
          formatter: cell => `${cell.getValue()}%` },
        { title: 'Current', field: 'current', widthGrow: 1, hozAlign: 'right',
          formatter: cell => `${cell.getValue()}A` },
        { title: 'Status', field: 'status', widthGrow: 1, hozAlign: 'center',
          formatter: cell => {
              const status = cell.getValue();
              const colors = { active: '#22c55e', inactive: '#ef4444', standby: '#f59e0b' };
              return `<span style="color:${colors[status]}">${status}</span>`;
          }
        }
    ],
    optionBuilder: getTableOption
};
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `selector` | string | O | 테이블 컨테이너 선택자 |
| `columns` | array | O | Tabulator 컬럼 정의 배열 |
| `columns[].title` | string | O | 컬럼 헤더 텍스트 |
| `columns[].field` | string | O | 데이터 필드명 |
| `columns[].widthGrow` | number | X | 상대적 너비 비율 |
| `columns[].hozAlign` | string | X | 정렬 (left, center, right) |
| `columns[].formatter` | function | X | 셀 포맷터 함수 |
| `optionBuilder` | function | X | Tabulator 옵션 생성 함수 |

**옵션 빌더 패턴**:
```javascript
function getTableOption(config, data) {
    return {
        layout: 'fitColumns',
        responsiveLayout: 'collapse',
        height: 250,
        placeholder: 'No data found',
        initialSort: [{ column: 'power', dir: 'desc' }],
        columns: config.columns
    };
}
```

---

### 4. Template Config (팝업/템플릿)

**용도**: HTML 템플릿 ID 및 팝업 생성 시 콜백 설정

```javascript
this.templateConfig = {
    popup: 'popup-ups'  // <template id="popup-ups"> 에서 추출
};

this.popupCreatedConfig = {
    chartSelector: '.chart-container',    // 차트 초기화 대상
    tableSelector: '.table-container',    // 테이블 초기화 대상
    events: {
        click: {
            '.close-btn': () => this.hideDetail(),
            '.refresh-btn': () => this.refresh()
        }
    }
};
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `popup` | string | O | 추출할 template 요소 ID |
| `chartSelector` | string | X | ECharts 초기화할 컨테이너 |
| `tableSelector` | string | X | Tabulator 초기화할 컨테이너 |
| `events` | object | X | 이벤트 바인딩 설정 |

**팝업 생성 콜백 패턴**:
```javascript
function onPopupCreated({ chartSelector, tableSelector, events }) {
    chartSelector && this.createChart(chartSelector);
    tableSelector && this.createTable(tableSelector);
    events && this.bindPopupEvents(events);
}
```

---

### 5. Summary Config (카드/대시보드)

**용도**: 요약 카드 렌더링 설정

```javascript
const statsConfig = [
    { key: 'visitors', label: 'Visitors', icon: '👥', format: v => v.toLocaleString() },
    { key: 'pageViews', label: 'Page Views', icon: '📄', format: v => v.toLocaleString() },
    { key: 'sessions', label: 'Sessions', icon: '🔗', format: v => v.toLocaleString() },
    { key: 'bounceRate', label: 'Bounce Rate', icon: '↩️', format: v => `${v}%` }
];
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `key` | string | O | 데이터 필드명 |
| `label` | string | O | 표시 라벨 |
| `icon` | string | X | 아이콘 (이모지 또는 클래스) |
| `format` | function | X | 값 포맷터 함수 |

**렌더 함수 패턴**:
```javascript
statsConfig.forEach(config => {
    const clone = template.content.cloneNode(true);
    const card = clone.querySelector('.stat-card');
    const icon = clone.querySelector('.stat-icon');
    const label = clone.querySelector('.stat-label');
    const value = clone.querySelector('.stat-value');

    card.dataset.statKey = config.key;
    icon.textContent = config.icon;
    label.textContent = config.label;
    value.textContent = config.format(data[config.key]);

    container.appendChild(clone);
});
```

---

### 6. Log Config (로그 뷰어)

**용도**: 로그 필드 매핑 및 제한 설정

```javascript
const config = {
    titleKey: 'title',
    logsKey: 'logs',
    logFields: {
        time: 'timestamp',
        level: 'severity',
        message: 'content'
    },
    maxLogs: 100
};
```

**속성 설명**:
| 속성 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `titleKey` | string | O | 제목 필드명 |
| `logsKey` | string | O | 로그 배열 필드명 |
| `logFields` | object | O | 개별 로그 항목 필드 매핑 |
| `logFields.time` | string | O | 시간 필드 |
| `logFields.level` | string | O | 레벨 필드 (info, warn, error) |
| `logFields.message` | string | O | 메시지 필드 |
| `maxLogs` | number | X | 최대 로그 수 (초과 시 오래된 것 제거) |

---

### 7. Icon/Status Config (상태 매핑)

**용도**: 상태값에 따른 아이콘/색상 매핑

```javascript
this.typeIcons = {
    ups: '⚡',
    pdu: '🔌',
    crac: '❄️',
    sensor: '🌡️'
};

this.statusColors = {
    normal: '#22c55e',
    warning: '#f59e0b',
    critical: '#ef4444',
    offline: '#6b7280'
};
```

---

## 바인딩 패턴

### 1. 기본 바인딩

```javascript
this.renderData = renderData.bind(this, config);
```

### 2. 다중 Config 조합

```javascript
// 배열 스프레드로 Config 병합
this.renderUPSInfo = renderUPSInfo.bind(this, [...this.baseInfoConfig, ...this.upsInfoConfig]);
```

### 3. 커링 + 바인딩

```javascript
// fx.curry를 사용한 커링
this.renderChart = fx.curry(renderLineData)(config).bind(this);
```

### 4. 옵션 빌더 분리

```javascript
// Config에 optionBuilder 포함
this.chartConfig = {
    xKey: 'timestamps',
    series: [...],
    optionBuilder: getMultiLineChartOption
};

// 렌더 시 분리
function renderChart(config, data) {
    const { optionBuilder, ...chartConfig } = config;
    const option = optionBuilder(chartConfig, data);
    this.updateChart('.chart-container', option);
}
```

---

## Config vs 하드코딩 기준

### Config로 분리해야 하는 것

| 항목 | 이유 |
|------|------|
| API 필드명 | 백엔드 변경에 대응 |
| DOM 선택자 | HTML 구조 변경에 대응 |
| 시리즈 정의 | 차트 추가/삭제 용이 |
| 컬럼 정의 | 테이블 구성 변경 용이 |
| 상태 매핑 | 비즈니스 규칙 변경 대응 |

### 하드코딩해도 되는 것

| 항목 | 이유 |
|------|------|
| 색상 팔레트 | 프로젝트 전체 스타일 |
| 폰트 크기 | 디자인 시스템 일관성 |
| 그리드 여백 | ECharts 기본 레이아웃 |
| 애니메이션 설정 | 공통 UX |

---

## TBD 패턴 (미정 필드)

컴포넌트를 API 없이 미리 개발할 때 사용하는 플레이스홀더:

```javascript
const config = {
    titleKey: 'TBD_title',        // 실제 API 필드명 미정
    logsKey: 'TBD_logs',
    logFields: {
        time: 'TBD_time',
        level: 'TBD_level',
        message: 'TBD_message'
    }
};

this.subscriptions = {
    TBD_topicName: ['renderData']  // 실제 topic 미정
};

this.customEvents = {
    click: {
        '.btn-clear': '@TBD_clearClicked'  // 실제 이벤트명 미정
    }
};
```

**TBD 필드 완성 시점**: 페이지 통합 또는 API 연동 시

---

## 실제 적용 사례

### Case 1: UPS 컴포넌트 (ECO 프로젝트)

```javascript
// 1. 데이터 Config
this.baseInfoConfig = [
    { key: 'name', selector: '.ups-name' },
    { key: 'zone', selector: '.ups-zone' },
    { key: 'status', selector: '.ups-status', dataAttr: 'status' }
];

this.upsInfoConfig = [
    { key: 'load', selector: '.ups-load', suffix: '%' },
    { key: 'batteryLevel', selector: '.ups-battery', suffix: '%' },
    // ...
];

// 2. 차트 Config
this.chartConfig = {
    xKey: 'timestamps',
    series: [
        { yKey: 'load', name: 'Load', color: '#3b82f6', smooth: true, areaStyle: true },
        { yKey: 'battery', name: 'Battery', color: '#22c55e', smooth: true }
    ],
    optionBuilder: getMultiLineChartOption
};

// 3. 템플릿 Config
this.templateConfig = { popup: 'popup-ups' };

// 4. 팝업 생성 Config
this.popupCreatedConfig = {
    chartSelector: '.chart-container',
    events: { click: { '.close-btn': () => this.hideDetail() } }
};

// 5. 바인딩
this.renderUPSInfo = renderUPSInfo.bind(this, [...this.baseInfoConfig, ...this.upsInfoConfig]);
this.renderChart = renderChart.bind(this, this.chartConfig);
```

### Case 2: TimeTrendChart (HANA_BANK_HIT_Dev)

```javascript
// 단일 Config로 전체 차트 정의
const config = {
    xKey: 'tm',
    seriesMap: [
        { key: 'val_max', name: '역대픽', color: '#526FE5' },
        { key: 'val_year', name: '연중최고픽', color: '#52BEE5' },
        { key: 'val_month', name: '월픽', color: '#009178' },
        { key: 'val_prev', name: '전일', color: '#52E5C3' },
        { key: 'val_today', name: '금일', color: '#AAFD84' }
    ],
    smooth: true,
    symbol: 'none',
    areaStyle: true,
    areaGradient: true,
    yAxis: { min: 0, max: 1800, interval: 600 }
};

// 커링으로 바인딩
this.renderChart = fx.curry(renderLineData)(config).bind(this);
```

---

## 요약

| Config 유형 | 주요 용도 | 핵심 속성 |
|-------------|----------|-----------|
| **Field Config** | API → DOM 매핑 | key, selector, suffix |
| **Chart Config** | ECharts 설정 | xKey, series, optionBuilder |
| **Table Config** | Tabulator 설정 | columns, optionBuilder |
| **Template Config** | 팝업 템플릿 | popup, events |
| **Summary Config** | 카드 렌더링 | key, label, icon, format |
| **Log Config** | 로그 뷰어 | logFields, maxLogs |
| **Status Config** | 상태 매핑 | typeIcons, statusColors |

**핵심 원칙**:
1. **Config는 What**, **렌더 함수는 How**
2. **외부 인터페이스(API 필드, 선택자)는 Config로 분리**
3. **TBD 패턴으로 API 없이 미리 개발 가능**
4. **optionBuilder로 복잡한 옵션 생성 로직 분리**

---

*최종 업데이트: 2025-12-31*
