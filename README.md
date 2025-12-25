# FIGMA_INTEGRATED

<<<<<<< HEAD
Figma 디자인을 RENOBIT 웹 빌더 런타임 컴포넌트로 변환하는 파이프라인입니다.

---

## Claude Code와 함께 사용하기

이 프로젝트는 **Claude Code**와 함께 사용하도록 설계되었습니다.

### CLAUDE.md 방식 vs Agent Skill 방식

| 항목 | CLAUDE.md (현재) | Agent Skill |
|------|------------------|-------------|
| **로드 시점** | 항상 (세션 시작 시) | 필요할 때만 |
| **발견 방식** | 파일 경로 (프로젝트 내) | description 매칭 |
| **사용 범위** | 이 프로젝트 안에서만 | 어디서든 (설치 시) |
| **콘텍스트 비용** | 높음 (항상 포함) | 낮음 (선택적) |

**현재 이 프로젝트는 CLAUDE.md 방식을 사용합니다.**

```
사용 흐름:
1. 프로젝트 clone
2. Claude Code 실행
3. Claude가 CLAUDE.md 자동 로드 → 컨텍스트 파악 완료
4. "Figma 링크 줄게" → 변환 작업 수행
5. 결과 도출
```

---

## 사전 준비 (필수)

이 프로젝트를 사용하기 전에 다음을 준비해야 합니다.

### 1. Figma Desktop 앱 설치

> **중요**: 웹 버전(figma.com)이 아닌 **데스크톱 앱**이 필요합니다.

1. [Figma Desktop 다운로드](https://www.figma.com/downloads/)
2. 설치 후 Figma 계정으로 로그인
3. 변환할 Figma 파일 열기

```
✅ Figma Desktop 앱 실행 → MCP 작동
❌ Figma 웹 버전 → MCP 작동 안 함
```

### 2. Figma Dev Mode 활성화

Figma Desktop에서 변환할 파일을 열고:

1. 우측 상단 `</>` 버튼 클릭 (또는 Shift + D)
2. Dev Mode 활성화 확인

### 3. Claude Code 설치

Claude Code가 설치되어 있어야 합니다.

```bash
# Claude Code 설치 확인
claude --version
```

설치가 안 되어 있다면: [Claude Code 설치 가이드](https://claude.ai/code)

### 4. 프로젝트 Clone 및 의존성 설치

```bash
# 1. 프로젝트 클론 (서브모듈 포함)
git clone --recursive <repository-url>
cd FIGMA_INTEGRATED

# 2. Figma_Conversion 의존성 설치
cd Figma_Conversion
npm install

# 3. Playwright 브라우저 설치 (스크린샷용)
npx playwright install chromium

# 4. 루트로 돌아가기
cd ..
```

### 5. Figma MCP 서버 등록

Figma Desktop이 실행 중인 상태에서:

```bash
# Figma MCP 서버 등록
claude mcp add figma-desktop --transport http --url http://127.0.0.1:3845/mcp

# 등록 확인
claude mcp list

# 예상 결과:
# figma-desktop: ✓ Connected
```

> **참고**: Figma Desktop이 실행 중이어야 MCP 서버에 연결됩니다.

---

## 사전 준비 체크리스트

```
[ ] Figma Desktop 앱 설치 및 실행
[ ] Figma Dev Mode 활성화
[ ] Claude Code 설치
[ ] 프로젝트 clone 완료
[ ] npm install 완료 (Figma_Conversion/)
[ ] Playwright 브라우저 설치 완료
[ ] Figma MCP 서버 등록 완료 (claude mcp add)
```

---

## 사용 방법

### 기본 워크플로우

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. Figma_Conversion                                                 │
│                                                                      │
│     Figma MCP로 디자인 정보 추출 → HTML/CSS 생성 → 스크린샷 검증        │
│                                                                      │
│     ❌ 스크립트 작업 없음 (순수 퍼블리싱)                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼  정적 HTML/CSS 전달
┌─────────────────────────────────────────────────────────────────────┐
│  2. RNBT_architecture                                                │
│                                                                      │
│     정적 HTML/CSS → 동적 컴포넌트 변환 + 런타임 구성                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 1: Figma → 정적 HTML/CSS

1. Figma Desktop에서 변환할 요소 선택
2. Claude Code 실행
3. Figma 링크 제공

```
사용자: "이 Figma 링크를 HTML/CSS로 변환해줘"
https://www.figma.com/design/VNqtXrH6ydqcDgYBsVFLbg/...?node-id=25-1393

Claude: (MCP로 데이터 추출 → HTML/CSS 생성 → 스크린샷 검증)
```

결과물 위치:
```
Figma_Conversion/Conversion/[프로젝트명]/[컴포넌트명]/
├── assets/                 # SVG, 이미지 에셋
├── screenshots/            # 구현 스크린샷
├── [컴포넌트명].html
└── [컴포넌트명].css
```

### Step 2: 정적 → 동적 컴포넌트

1. 정적 HTML/CSS를 RNBT_architecture로 이동
2. 스크립트 추가 (register.js, destroy.js)
3. 런타임 구성

> **참고**: 동적 컴포넌트는 페이지가 오케스트레이션합니다. 컴포넌트 외에 **페이지 스크립트**(before_load, loaded, before_unload)와 **Mock Server** 작업이 필요할 수 있습니다. 자세한 내용은 `RNBT_architecture/README.md`를 참조하세요.

```
사용자: "이 정적 컴포넌트를 동적 컴포넌트로 변환해줘"

Claude: (라이프사이클 구성 → 데이터 바인딩 → 이벤트 처리)
```

결과물 위치:
```
RNBT_architecture/Projects/[프로젝트명]/page/components/[컴포넌트명]/
├── [컴포넌트명].html
├── [컴포넌트명].css
├── register.js             # 컴포넌트 등록 스크립트
├── destroy.js              # 컴포넌트 정리 스크립트
└── preview.html            # 미리보기 (서버 없이 브라우저에서 직접 확인 가능)
```

---

## 프로젝트 구조

```
FIGMA_INTEGRATED/
├── CLAUDE.md                   # Claude Code 지침서 (자동 로드)
├── README.md                   # 이 문서
├── index.html                  # 프로젝트 포탈 페이지
├── discussions/                # 설계 논의 문서
│
├── Figma_Conversion/           # 서브모듈 1: Figma → 정적 HTML/CSS
│   ├── CLAUDE.md               # Figma 변환 상세 지침
│   ├── Conversion/             # 변환 결과물
│   └── package.json            # 의존성 (playwright)
│
└── RNBT_architecture/          # 서브모듈 2: 정적 → 동적 컴포넌트
    ├── CLAUDE.md               # RNBT 작업 지침
    ├── README.md               # 아키텍처 가이드
    ├── Utils/                  # 공용 유틸리티
    ├── Examples/               # 예제 프로젝트
    └── Projects/               # 실제 프로젝트
=======
> Figma Design to Dynamic Component Workflow

## 2025년 12월 현재: RENOBIT 프로젝트 스탠다드 구축

---

## Why?

프로젝트마다 만들어지는 결과물이 다르고, 페이지와 컴포넌트의 결과들이 섞여 있습니다.

### Problem

- 프로젝트를 유지하고 보수하는 비용이 증가합니다.
- 프로젝트에서 만든 결과물을 다시 사용하기 어렵습니다.

---

## So We Focused

엄격한 영역(페이지, 컴포넌트) 분리를 통해:

- **결과물의 일관성** 추구
- **컴포넌트 재사용성** 확보

### 기대 효과

- 유지보수 비용 절감
- 같은 작업 최소화로 다음 프로젝트 효율성 향상

---

## How?

### 아키텍처 패턴

| 패턴 | 설명 |
|------|------|
| **PUB-SUB** | GlobalDataPublisher를 통한 데이터 구독/발행 |
| **Event-Driven** | WEventBus를 통한 컴포넌트 간 통신 |

### 자체 개발 라이브러리

| 라이브러리 | 역할 |
|------------|------|
| **GlobalDataPublisher** | 페이지 레벨 데이터 관리 및 구독 |
| **WEventBus** | 이벤트 기반 컴포넌트 통신 |
| **WKit.js** | 유틸리티 함수 모음 |
| **Mixin** | 컴포넌트 기능 확장 패턴 (FreeCodeMixin, ModelLoaderMixin 등) |
| **Fx.js** | 함수형 프로그래밍 라이브러리 |

### 통합 가이드 문서

RENOBIT을 활용하는 파트너라면 누구나 봐야 할 문서:

- [RNBT_architecture](https://github.com/JSKIM-90/RNBT_architecture)

---

## AI + Tool 활용 Workflow

> 패턴은 좋은데... 작업이 너무 어렵지 않나?
> 결국 컴포넌트도 직접 만들고 페이지도 일일이 만들어야해?

**그래서, AI와 외부 Tool을 활용한 Workflow 프로세스 확립**

### 사용 도구

| 도구 | 역할 |
|------|------|
| **Claude Code** | AI 기반 코드 생성 및 리팩토링 |
| **Figma MCP Server** | Figma Desktop과 Claude Code 연동 |
| **Playwright** | 구현 결과 스크린샷 캡처 및 비교 |

### End-to-End Workflow

```
Figma Design → Figma MCP → Static HTML/CSS → Dynamic Component → Preview
```

### Agent Skills 관점

> **Agent Skills**는 Claude의 기능을 확장하는 모듈식 기능입니다. 각 Skill은 지침, 메타데이터 및 선택적 리소스(스크립트, 템플릿)를 패키징하며, Claude는 관련이 있을 때 자동으로 이를 사용합니다.
> — [Claude Agent Skills](https://platform.claude.com/docs/ko/agents-and-tools/agent-skills/overview)

이 저장소는 Agent Skills 아키텍처와 유사한 구조를 가집니다.

| Agent Skills | FIGMA_INTEGRATED |
|--------------|------------------|
| SKILL.md (메타데이터) | README.md (진입점) |
| 지침 (워크플로우) | CLAUDE.md (작업 규칙) |
| 리소스 (코드, 참조) | 서브모듈 (Figma_Conversion, RNBT_architecture) |

**점진적 공개**: Claude는 필요할 때만 서브모듈의 상세 문서를 로드하여 컨텍스트를 효율적으로 사용합니다.

---

## Phase 1: Design to Static HTML/CSS

Figma 디자인을 정적 HTML/CSS로 변환하는 워크플로우.

### 관련 문서

- [Figma_Conversion/CLAUDE.md](Figma_Conversion/CLAUDE.md) - Figma → HTML/CSS 변환 규칙

### 주요 기능

- Figma MCP를 통한 디자인 메타데이터 추출
- Playwright를 통한 스크린샷 비교 검증
- 컴포넌트 단위 HTML/CSS 생성

---

## Phase 2: Static to Dynamic Component

정적 HTML/CSS를 동적 컴포넌트로 변환하는 워크플로우.

### 관련 문서

- [RNBT_architecture/README.md](RNBT_architecture/README.md) - 런타임 프레임워크 설계

---

## 저장소 구조

```
FIGMA_INTEGRATED/
├── Figma_Conversion/     # (서브모듈) Figma → Static HTML/CSS
├── RNBT_architecture/    # (서브모듈) Static → Dynamic Component
└── index.html            # 프로젝트 포털
>>>>>>> 55fe9e1c8c4c8a55700b6d9ab698b3dafcc58e3a
```

---

<<<<<<< HEAD
## 서브모듈 문서

### Figma_Conversion

- **작업 지침**: `Figma_Conversion/CLAUDE.md`
- **컴포넌트 구조**: `Figma_Conversion/PUBLISHING_COMPONENT_STRUCTURE.md`

### RNBT_architecture

- **작업 지침**: `RNBT_architecture/CLAUDE.md`
- **설계 문서**: `RNBT_architecture/README.md`

---

## MCP 서버 동작 원리

```
Figma 디자인 ←→ MCP 서버 ←→ Claude Code ←→ HTML/CSS 코드
```

### MCP 제공 도구

| 도구 | 역할 | 출력 |
|------|------|------|
| `get_metadata` | 크기/위치 정보 | `{width, height, x, y}` |
| `get_code` | 코드 생성 | HTML/CSS 코드 |
| `get_image` | 스크린샷 | PNG 이미지 |
| `get_variable_defs` | 디자인 토큰 | 색상, 간격, 폰트 변수 |

### 왜 MCP가 필요한가?

- ❌ **수동 작업 없이**: 크기, 색상, 간격을 일일이 측정할 필요 없음
- ✅ **정확한 구현**: Figma 디자인의 정확한 수치를 그대로 사용
- ⚡ **빠른 개발**: 디자인 → 코드 변환 시간 단축

---

## 문제 해결

### MCP 연결 안 됨

```bash
# 1. Figma Desktop이 실행 중인지 확인
# 2. MCP 서버 상태 확인
claude mcp list

# 3. 연결 안 되면 재등록
claude mcp remove figma-desktop
claude mcp add figma-desktop --transport http --url http://127.0.0.1:3845/mcp
```

### Playwright 스크린샷 실패

```bash
# 브라우저 재설치
npx playwright install chromium
```

### 서브모듈 누락

```bash
# 서브모듈 초기화 및 업데이트
git submodule update --init --recursive
=======
## Quick Start

```bash
# 서브모듈 초기화
git submodule init
git submodule update

# Figma Conversion 작업
cd Figma_Conversion
npm install
npm run serve

# RNBT architecture 작업
cd ../RNBT_architecture/Projects/[프로젝트명]/mock_server
npm install
npm start  # port 3000
>>>>>>> 55fe9e1c8c4c8a55700b6d9ab698b3dafcc58e3a
```

---

<<<<<<< HEAD
## 대상 사용자

이 프로젝트는 다음 환경에서 사용하도록 설계되었습니다:

- **RENOBIT 웹 빌더** 사용자
- **Figma** 디자인 사용자
- **Claude Code** 사용자

> 다른 웹 빌더를 사용하는 경우, RNBT_architecture 부분을 해당 웹 빌더에 맞게 수정해야 합니다.

---

## 라이선스

[라이선스 정보]

---

*최종 업데이트: 2025-12-26*
=======
## 2026년 방향

2025년에 컴포넌트 생산 프로세스를 확립했다면, 2026년은 **확장과 안정화**입니다.

- **Template 사전 생산**: 재사용 가능한 Template 라이브러리 구축
- **Template Managing System**: 생산된 Template의 체계적 관리
- **코드베이스 안정화**: 확장성을 위한 기반 정비
- **Spring 6 업그레이드**: 백엔드 프레임워크 현대화

상세 내용: [discussions/ROADMAP_2026.md](discussions/ROADMAP_2026.md)

---

## 관련 링크

- [FIGMA_INTEGRATED](https://github.com/JSKIM-90/FIGMA_INTEGRATED) - 이 저장소
- [RNBT_architecture](https://github.com/JSKIM-90/RNBT_architecture) - 런타임 프레임워크
- [Figma_Conversion](https://github.com/JSKIM-90/FIGMA_CONVERSION) - Figma 변환 도구

---

*최종 업데이트: 2025-12-23*
>>>>>>> 55fe9e1c8c4c8a55700b6d9ab698b3dafcc58e3a
