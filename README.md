# FIGMA_INTEGRATED

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
```

---

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
```

---

## 관련 링크

- [FIGMA_INTEGRATED](https://github.com/JSKIM-90/FIGMA_INTEGRATED) - 이 저장소
- [RNBT_architecture](https://github.com/JSKIM-90/RNBT_architecture) - 런타임 프레임워크
- [Figma_Conversion](https://github.com/JSKIM-90/FIGMA_CONVERSION) - Figma 변환 도구

---

*최종 업데이트: 2025-12-23*
