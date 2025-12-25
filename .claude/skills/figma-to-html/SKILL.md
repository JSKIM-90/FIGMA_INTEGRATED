---
name: figma-to-html
description: Figma 디자인을 정적 HTML/CSS로 변환합니다. Figma 링크나 node-id가 제공되면 MCP를 통해 디자인 정보를 가져와 정확한 코드를 생성합니다. Use when converting Figma designs to HTML/CSS, or when user mentions Figma links, design conversion, or component implementation.
---

# Figma to HTML/CSS 변환

Figma 디자인을 정적 HTML/CSS로 변환하는 Skill입니다.

## 사전 조건

- Figma Desktop 앱 실행 중
- 대상 Figma 파일이 열려 있음
- Figma MCP 서버 등록됨 (`claude mcp add figma-desktop --transport http --url http://127.0.0.1:3845/mcp`)

## 워크플로우

```
1. Figma 링크에서 node-id 추출 (25-1393 → 25:1393)
2. MCP 도구 호출
   - get_design_context (dirForAssetWrites 필수)
   - get_screenshot (원본 비교용)
3. Tailwind → 순수 CSS 변환
4. HTML/CSS 파일 생성
5. Playwright 스크린샷 캡처
6. 시각적 비교 및 수정
```

## MCP 도구 사용

### 디자인 정보 + 에셋 다운로드
```javascript
mcp__figma-desktop__get_design_context({
  nodeId: "781:47496",
  clientLanguages: "html,css",
  clientFrameworks: "vanilla",
  dirForAssetWrites: "./Conversion/[프로젝트명]/[컴포넌트명]/assets"
})
```

### 원본 스크린샷 (비교용)
```javascript
mcp__figma-desktop__get_screenshot({ nodeId: "781:47496" })
```

## 출력 폴더 구조

```
Conversion/
└── [프로젝트명]/
    └── [컴포넌트명]/
        ├── assets/           # SVG, 이미지 에셋 (자동 다운로드)
        ├── screenshots/      # 구현물 스크린샷
        │   └── impl.png
        ├── [컴포넌트명].html
        └── [컴포넌트명].css
```

## 핵심 규칙

### 추측 금지 원칙
- MCP가 제공하는 정확한 데이터 사용
- "비슷해 보인다" ≠ "일치한다"
- 차이점 발견 시 MCP 데이터 다시 확인

### 에셋 규칙
- `dirForAssetWrites`로 에셋 자동 다운로드
- 로컬 경로 사용 (`./assets/파일명.svg`)
- `http://127.0.0.1:3845/...` URL 직접 사용 금지

### 컨테이너 구조
```html
<div id="component-container">
  <!-- Figma 내부 구조 그대로 -->
</div>
```
- 컨테이너 크기 = Figma 선택 요소 크기
- `overflow: auto` 추가 (동적 렌더링 대응)

### box-sizing 주의
```css
.element {
  box-sizing: border-box;
  height: 24px;  /* Figma에서 측정한 높이 명시 */
  border: 1px solid #000;
}
```

## Playwright 스크린샷

```javascript
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({
    viewport: { width: 524, height: 350 }  // Figma 프레임 크기
  });
  await page.goto('http://localhost:3000/path/to/component.html');
  await page.screenshot({ path: './screenshots/impl.png' });
  await browser.close();
})();
"
```

## 완료 체크리스트

- [ ] 전체 컨테이너 크기가 Figma와 일치
- [ ] gap 값 정확 (row-gap ≠ column-gap 체크)
- [ ] 헤더-컨텐츠 간격 일치
- [ ] 상단/하단 여백 일치
- [ ] border 있는 요소에 height 명시
- [ ] 에셋 로컬 경로로 변환
- [ ] Playwright 스크린샷과 Figma 원본 비교 완료
