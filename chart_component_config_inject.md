# 차트 컴포넌트 Config 주입 아키텍처

## 목적

**Line 차트면 Line용 renderData 하나로 충분하다.**

차트 타입별로 공통 렌더 함수를 만들고, Raw API 필드를 매핑하는 key만 Config로 주입하여 재사용한다.

---

## 핵심 통찰

### Config의 key는 Raw API 필드명이다

Config로 받을 내용은 **Raw API 응답에서 값을 꺼낼 key**다.

```javascript
// Raw API 응답
[
    { tm: "00", val_max: 1200, val_year: 1100, val_month: 600 },
    { tm: "02", val_max: 1300, val_year: 1150, val_month: 650 },
    ...
]

// Config: API 응답의 어떤 key를 사용할지 선언
const config = {
    xKey: 'tm',           // X축에 사용할 API 필드
    seriesMap: [
        { key: 'val_max', name: '역대픽', color: '#526FE5' },
        { key: 'val_year', name: '연중최고픽', color: '#52BEE5' }
    ]
};
```

- `key`: Raw API 필드명 (데이터마다 다름)
- `name`: 차트에 표시할 이름 (디자인/기획)
- `color`: 색상 (디자인)

### 차트는 화면에 배치되는 순간 역할이 정해진다

**중요한 원칙:** 차트가 사용할 key까지 동적이어서는 안 된다.

- 화면을 구성할 때 배치된 차트가 어떨 때는 `date` key였다가, 어떨 때는 `index` key였다가 하지 않는다
- 가능은 하겠으나 **일반적이지 않다**
- "시간대별 거래량 차트"는 항상 `tm`이 X축
- 이 매핑은 런타임에 바뀔 이유가 없다

```
Config (정적, 배치 시점)     Data (동적, 런타임)
─────────────────────────   ─────────────────────
xKey: 'tm'                  response.data
seriesMap: [...]
```

- **Config**: 차트의 역할/정체성 정의 → 배치 시점에 고정
- **Data**: 차트에 채워질 값 → Pub/Sub으로 동적 주입

---

## 패턴: 커링으로 Config/Data 분리

```javascript
// 커링 + 바인딩
this.renderChart = fx.curry(renderLineData)(config).bind(this);

// Pub/Sub 구독
this.subscriptions = {
    timeTrendData: ['renderChart']
};

// 렌더 함수 (호이스팅)
function renderLineData(config, response) {
    const { data } = response;
    // config.xKey, config.seriesMap으로 data에서 값 추출
}
```

### 왜 이렇게 하는가

1. **renderData 재사용**: Line 차트용 함수 하나로 여러 컴포넌트 커버
2. **Config는 정적, Data는 동적**: 생명주기가 다른 두 인자를 커링으로 분리
3. **기존 아키텍처 유지**: `subscriptions`, `bind(this)` 패턴 그대로

### function 선언을 사용하는 이유

```javascript
// 위에서부터 "뭘 하는지" 먼저 보임
this.renderChart = fx.curry(renderLineData)(config).bind(this);

this.subscriptions = { ... };

// "어떻게 하는지"는 아래에서 확인 (호이스팅됨)
function renderLineData(config, response) { ... }
```

---

## 실제 구현 예시: TimeTrendChart

### Config 정의

```javascript
const config = {
    // Raw API 필드 매핑
    xKey: 'tm',             // X축에 사용할 API 필드

    // 시리즈 정의: key는 API 필드명, name은 범례 표시명
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

### 전체 register.js 구조

```javascript
const { subscribe } = GlobalDataPublisher;
const { bindEvents } = WKit;
const { each, curry } = fx;

// ======================
// CONFIG (정적 선언)
// ======================

const config = { /* 위 참조 */ };

// ======================
// BINDINGS (커링 + 바인딩)
// ======================

this.renderChart = curry(renderLineData)(config).bind(this);

// ======================
// SUBSCRIPTIONS
// ======================

this.subscriptions = {
    timeTrendData: ['renderChart']
};

fx.go(
    Object.entries(this.subscriptions),
    each(([topic, fnList]) =>
        each(fn => this[fn] && subscribe(topic, this, this[fn]), fnList)
    )
);

// ======================
// INITIALIZE ECHARTS
// ======================

const chartContainer = this.element.querySelector('#echarts');
this.chartInstance = echarts.init(chartContainer);

this.resizeObserver = new ResizeObserver(() => {
    this.chartInstance && this.chartInstance.resize();
});
this.resizeObserver.observe(chartContainer);

// ======================
// EVENT BINDING
// ======================

this.customEvents = {
    click: { '.btn': '@filterClicked' }
};

bindEvents(this, this.customEvents);

// ======================
// RENDER FUNCTION (호이스팅)
// ======================

function renderLineData(config, response) {
    const { data } = response;
    if (!data || !Array.isArray(data)) return;

    const { xKey, seriesMap, smooth, symbol, areaStyle, areaGradient, yAxis } = config;

    const option = {
        xAxis: {
            type: 'category',
            data: fx.go(data, fx.map(d => d[xKey]))
        },
        yAxis: {
            type: 'value',
            min: yAxis.min,
            max: yAxis.max,
            interval: yAxis.interval
        },
        series: fx.go(
            seriesMap,
            fx.map(s => ({
                name: s.name,
                type: 'line',
                smooth,
                symbol,
                data: fx.go(data, fx.map(d => d[s.key])),
                itemStyle: { color: s.color },
                lineStyle: { color: s.color, width: 2 },
                areaStyle: areaStyle ? {
                    color: areaGradient ? {
                        type: 'linear',
                        x: 0, y: 0, x2: 0, y2: 1,
                        colorStops: [
                            { offset: 0, color: s.color + '80' },
                            { offset: 1, color: s.color + '10' }
                        ]
                    } : s.color
                } : undefined
            }))
        )
    };

    try {
        this.chartInstance.setOption(option);
    } catch (e) {
        console.error('[TimeTrendChart] setOption error:', e);
    }
}
```

---

## Config vs 하드코딩 기준

| 항목 | Config | 하드코딩 | 이유 |
|------|--------|----------|------|
| `xKey` | ✅ | | Raw API 필드명, 데이터마다 다름 |
| `seriesMap[].key` | ✅ | | Raw API 필드명, 데이터마다 다름 |
| `seriesMap[].name` | ✅ | | 범례 표시명, 기획에 따라 다름 |
| `seriesMap[].color` | ✅ | | 디자인에 따라 다름 |
| `smooth`, `symbol` | ✅ | | 차트 스타일, 변경 가능 |
| `yAxis.min/max` | ✅ | | 데이터 범위에 따라 다름 |
| `type: 'line'` | | ✅ | Line 렌더 함수니까 고정 |
| `tooltip.trigger` | | ✅ | Line 차트는 항상 'axis' |

---

## 요약

| 시점 | 무엇을 | 어떻게 |
|------|--------|--------|
| 배치 | Raw API 필드 매핑 | Config 객체 정의 |
| 초기화 | 커링 + 바인딩 | `fx.curry(renderFn)(config).bind(this)` |
| 런타임 | 데이터만 전달 | Pub/Sub → `renderChart(response)` |

**Config는 정적 선언, Data는 동적 주입** — 이것이 본질이다.

---

## 작업 중 실수 기록

| 실수 | 원인 | 교훈 |
|------|------|------|
| `fx.prop` 없는 함수 사용 | "함수형 라이브러리에 흔히 있으니까" 추측 | **실제 코드 확인 없이 가정하지 말 것** |
| `bindMethods` 없는 함수 언급 | 아키텍처 코드 읽기 전에 추측 | **코드를 먼저 읽고 분석할 것** |

---

## 현재 상태 및 한계

- **패턴 적용 예시** 수준 (프로덕션 완성 아님)
- `renderLineData`가 아직 컴포넌트 내부에 있음 (공통 모듈 분리 필요)
- 실제 Raw API 데이터로 테스트 안 됨
