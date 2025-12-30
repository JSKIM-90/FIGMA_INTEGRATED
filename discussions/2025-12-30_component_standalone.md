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

## 결론

**API나 Figma 없이 개발 가능한 것:**
- HTML/CSS 구조
- 이벤트 정의 (customEvents)
- 렌더 함수 로직
- beforeDestroy 정리 로직
- preview.html로 Mock 테스트

**API 확정 시 수정할 것:**
- `subscriptions`의 topic명
- `config`의 key (API 필드명)

---

*작성일: 2025-12-30*
