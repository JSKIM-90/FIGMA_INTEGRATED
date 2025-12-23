---
name: html-to-component
description: 정적 HTML/CSS를 RNBT 동적 컴포넌트로 변환합니다. Figma Conversion에서 생성된 정적 파일을 RNBT_architecture 패턴에 맞게 동적 컴포넌트로 변환합니다. Use when converting static HTML to dynamic components, or when implementing RNBT components with data binding.
---

# HTML to Dynamic Component 변환

정적 HTML/CSS를 RNBT 동적 컴포넌트로 변환하는 Skill입니다.

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
├── views/component.html      # 데이터 바인딩 마크업
├── styles/component.css      # 스타일 (Figma CSS 기반)
├── scripts/
│   ├── register.js           # 초기화 + 메서드 정의
│   └── destroy.js            # 정리
└── preview.html              # 독립 테스트
```

## 변환 워크플로우

### 1. 정적 HTML 분석
```html
<!-- 정적 (Figma Conversion) -->
<div class="performance-status">
  <div class="value">1,234</div>
  <div class="label">TPS</div>
</div>
```

### 2. 데이터 바인딩 마크업 변환
```html
<!-- 동적 (RNBT) -->
<div class="performance-status">
  <div class="value" data-bind="tps"></div>
  <div class="label">TPS</div>
</div>
```

### 3. register.js 작성

```javascript
/*
 * [ComponentName] - RNBT Component
 */

initComponent.call(this);

function initComponent() {
  // 1. 데이터 구독
  this.subscription = GlobalDataPublisher.subscribe('topicName', (data) => {
    this.renderData(data);
  });

  // 2. 렌더링 함수
  this.renderData = function(data) {
    const el = this.element;
    el.querySelector('[data-bind="tps"]').textContent = data.tps;
  };

  console.log('[ComponentName] Registered:', this.id);
}
```

### 4. destroy.js 작성

```javascript
function onInstanceUnLoad() {
  if (this.subscription) {
    this.subscription.unsubscribe();
    this.subscription = null;
  }
  console.log('[ComponentName] Destroyed:', this.id);
}
```

## 핵심 패턴

### PUB-SUB (GlobalDataPublisher)
```javascript
// 구독
this.subscription = GlobalDataPublisher.subscribe('topic', callback);

// 구독 해제 (destroy.js에서)
this.subscription.unsubscribe();
```

### Event-Driven (WEventBus)
```javascript
// 이벤트 발행
WEventBus.emit('@eventName', { targetInstance: this, event: data });

// 이벤트 핸들러 (페이지에서)
this.eventBusHandlers = {
  '@eventName': ({ event, targetInstance }) => { /* ... */ }
};
```

### datasetInfo 패턴 (자기 완결 컴포넌트)
```javascript
this.datasetInfo = [
  { datasetName: 'api', param: { id: 123 }, render: ['renderMethod'] }
];
```

## 라이브러리 활용

| 라이브러리 | 용도 | 예시 |
|------------|------|------|
| **ECharts** | 차트 | `applyEChartsMixin(this)` |
| **Tabulator** | 테이블 | `applyTabulatorMixin(this)` |
| **Fx.js** | 함수형 처리 | `fx.go(data, fx.map(...), fx.each(...))` |

## preview.html 템플릿

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <link rel="stylesheet" href="./styles/component.css">
</head>
<body>
  <div id="component-container" style="width: 524px; height: 350px;">
    <!-- views/component.html 내용 -->
  </div>
  <script>
    // Mock 데이터로 테스트
    const mockData = { tps: "1,234" };
    // 렌더링 테스트
  </script>
</body>
</html>
```

## 체크리스트

- [ ] 정적 HTML 구조 분석 완료
- [ ] data-bind 속성 추가
- [ ] register.js 구독 패턴 구현
- [ ] destroy.js 정리 로직 구현
- [ ] preview.html 독립 테스트 작성
- [ ] Mock 데이터로 렌더링 확인

## 참고 문서

- [RNBT_architecture/README.md](../../RNBT_architecture/README.md)
- [Component Lifecycle](../../RNBT_architecture/README.md#컴포넌트-생명주기)
