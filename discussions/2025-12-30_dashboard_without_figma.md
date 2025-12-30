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
