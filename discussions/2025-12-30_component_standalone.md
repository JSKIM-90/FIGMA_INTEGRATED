# 컴포넌트 단독 개발

## 목적

API나 Figma 없이 컴포넌트는 어디까지 개발할 수 있는가?

---

## 컴포넌트 세트

```
components/MyComponent/
├── views/component.html
├── styles/component.css
├── scripts/
│   ├── register.js
│   └── beforeDestroy.js
└── preview.html
```

---

### views/component.html

```html
<div class="my-component">
    <span class="value1">-</span>
    <span class="value2">-</span>
    <button class="btn">Click</button>
</div>
```

### styles/component.css

```css
.my-component {
    padding: 16px;
    background: #1a1f2e;
    border-radius: 8px;
}

.my-component .value1,
.my-component .value2 {
    color: #e0e6ed;
}
```

### scripts/register.js

```javascript
const { subscribe } = GlobalDataPublisher;
const { bindEvents } = WKit;

// 1. 구독 정의
this.subscriptions = {
    TBD_topicName: ['renderData']
};

fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, fnList]) =>
        fx.each(fn => this[fn] && subscribe(topic, this, this[fn]), fnList)
    )
);

// 2. 이벤트 정의
this.customEvents = {
    click: {
        '.btn': '@buttonClicked'
    }
};

bindEvents(this, this.customEvents);

// 3. Config 정의
const config = {
    fields: [
        { key: 'TBD_field1', selector: '.value1' },
        { key: 'TBD_field2', selector: '.value2' }
    ]
};

// 4. 렌더 함수
this.renderData = renderData.bind(this, config);

function renderData(config, response) {
    const { data } = response;
    config.fields.forEach(({ key, selector }) => {
        const el = this.appendElement.querySelector(selector);
        if (el) el.textContent = data[key];
    });
}
```

### scripts/beforeDestroy.js

```javascript
const { unsubscribe } = GlobalDataPublisher;
const { removeCustomEvents } = WKit;

// 1. 구독 해제
fx.go(
    Object.entries(this.subscriptions),
    fx.each(([topic, _]) => unsubscribe(topic, this))
);

// 2. 이벤트 해제
removeCustomEvents(this, this.customEvents);

// 3. this.에 할당한 것들 null
this.subscriptions = null;
this.customEvents = null;
this.renderData = null;
```

### preview.html

```html
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="styles/component.css">
</head>
<body>
    <div id="component-root"></div>

    <script type="module">
        // Mock 컨텍스트
        const mockThis = {
            appendElement: document.getElementById('component-root')
        };

        // HTML 삽입
        mockThis.appendElement.innerHTML = `
            <div class="my-component">
                <span class="value1">-</span>
                <span class="value2">-</span>
                <button class="btn">Click</button>
            </div>
        `;

        // Mock 데이터로 렌더 테스트
        const config = {
            fields: [
                { key: 'field1', selector: '.value1' },
                { key: 'field2', selector: '.value2' }
            ]
        };

        const mockData = { field1: '25.5', field2: '60%' };

        config.fields.forEach(({ key, selector }) => {
            const el = mockThis.appendElement.querySelector(selector);
            if (el) el.textContent = mockData[key];
        });
    </script>
</body>
</html>
```

---

## 논의

**이슈**: API도 없고, 디자인도 없는데 컴포넌트를 얼마나 미리 만들 수 있는가?

**전제**:
- 컴포넌트를 미리 만들어놓고, 이후 페이지/프로젝트 작업 시 코드 수정 없이 쓸 수 있어야 함
- config 값 변경(topic명, key)은 기능을 건드리지 않으므로 코드 수정이 아님
- 함수는 데이터를 받아서 동작하고, 데이터 출처는 관심 두지 않아야 재사용 가능

**검토**:
- HTML/CSS 구조 → 마크업이므로 미리 정할 수 있음
- 셀렉터 → 마크업 기반이므로 미리 정할 수 있음
- 이벤트 타입/셀렉터 → 마크업 기반이므로 미리 정할 수 있음
- 렌더 함수 로직 → config를 받아 DOM에 바인딩하는 구조이므로 미리 완성 가능
- 이벤트 바인딩 로직 → bindEvents 호출이므로 미리 완성 가능
- beforeDestroy → register의 역순 정리이므로 미리 완성 가능
- topic명 → 외부 데이터 소스에 의존, 미리 정할 수 없음
- config key → API 필드명에 의존, 미리 정할 수 없음
- customEvents 이벤트명 → 페이지에서 받아 처리하므로 페이지와의 계약, 미리 정할 수 없음

---

## 결론

**미리 정할 수 없는 것: 외부 인터페이스**
- `subscriptions`의 topic명 (데이터 입력)
- `config`의 key (API 필드명)
- `customEvents`의 이벤트명 (이벤트 출력)

**미리 완성 가능한 것: 그 외 전부**
- HTML/CSS 구조
- 이벤트 바인딩 로직
- 렌더 함수 로직
- beforeDestroy 정리 로직
- preview.html로 Mock 테스트

---

*작성일: 2025-12-30*
