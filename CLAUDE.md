# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Repository Overview

이 저장소는 Figma 디자인을 웹 빌더 런타임 컴포넌트로 변환하는 두 개의 서브모듈로 구성됩니다.

| 서브모듈 | 역할 | Figma MCP |
|----------|------|-----------|
| **Figma_Conversion/** | Figma → 정적 HTML/CSS 추출 | 필요 |
| **RNBT_architecture/** | 정적 → 동적 컴포넌트 변환 + 런타임 | 불필요 |

---

## End-to-End Workflow

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

---

## 서브모듈 문서

각 서브모듈은 독립적으로 작업 가능하며, 상세 문서를 포함합니다:

### Figma_Conversion

- **작업 지침**: `Figma_Conversion/CLAUDE.md`
- **컴포넌트 구조**: `Figma_Conversion/PUBLISHING_COMPONENT_STRUCTURE.md`

### RNBT_architecture

- **작업 지침**: `RNBT_architecture/CLAUDE.md`
- **설계 문서**: `RNBT_architecture/README.md`
- **컴포넌트 구조**: `RNBT_architecture/Analysis/RUNTIME_COMPONENT_STRUCTURE.md`
- **Default JS 템플릿**: `RNBT_architecture/Analysis/DEFAULT_JS.md`
- **프로젝트 템플릿**: `RNBT_architecture/Analysis/PROJECT_TEMPLATE.md`

---

## Claude Code 출력 규칙

**긴 텍스트 출력 시 코드 블록 사용** - 터미널에서 긴 한글 텍스트나 마크다운이 제대로 표시되지 않는 문제가 있음

- 코드 블록 (` ``` `)으로 감싼 내용: 정상 출력됨
- 일반 마크다운/한글 텍스트: 출력이 잘릴 수 있음
- 따라서 긴 문서 내용을 보여줄 때는 코드 블록으로 감싸서 출력할 것

---

*최종 업데이트: 2025-12-19*
