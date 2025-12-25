# Agent Skill 변환 가능성 분석

**작성일**: 2025-12-26
**작성자**: Claude (분석)
**검토자**: 김준성

---

## 0. 논의 배경

### 목적

RENOBIT 웹 빌더를 사용하는 개발자가 Figma로부터 시작해서 프로젝트를 빠르고 효율적으로 구축하도록 돕는 것.

### 제공 방식 선택지

| 방식 | 설명 |
|------|------|
| A. 현재 형태 유지 | CLAUDE.md 기반, 프로젝트 clone 후 Claude Code와 함께 사용 |
| B. Agent Skill로 재구성 | .claude/skills/ 형태로 패키징하여 Claude Code 공식 형태로 제공 |

→ **B 방식 선택 이유**:
- 추가적인 지시 최소화 (개발자가 매번 설명할 필요 없음)
- Claude Code 공식 형태 (비공식 프로젝트가 아닌 표준 형태)

---

## 1. Agent Skill이란?

### 정의 (공식 문서 기반)

**"전문성(Expertise)을 발견 가능한 기능(Discoverable Capability)으로 패키징한 것"**

### 구조

```
my-skill/
├── SKILL.md (필수)         ← 지침 + 설정
├── reference.md (선택)     ← 상세 문서
├── scripts/                ← 실행 가능한 스크립트
│   ├── helper.py
│   └── validate.sh
└── templates/              ← 템플릿 파일
    └── template.txt
```

### 핵심 특성

| 특성 | 설명 |
|------|------|
| **Model-Invoked** | 사용자가 /command 입력 필요 없음. Claude가 맥락을 보고 자동으로 Skill 활성화 |
| **스크립트 실행** | scripts/ 폴더의 Python, Bash 등 실행 가능 |
| **도구 권한 제어** | allowed-tools로 특정 도구만 허용 가능 |
| **Hooks/MCP 통합** | PreToolUse, PostToolUse로 실행 전후 제어, MCP 서버 도구 호출 가능 |
| **팀 공유** | git commit으로 자동 배포, Plugin으로 패키징 가능 |

### Agent Skill의 목적 (3가지)

1. **특정 워크플로우를 위한 기능 확장** - Claude의 역량을 특정 작업에 맞게 확장
2. **팀 간 전문성 공유** - git을 통해 expertise 배포
3. **반복적인 프롬프팅 감소** - 매번 같은 지침 반복할 필요 제거

---

## 2. 초기 분석의 오류

### 잘못된 접근

| 오류 | 내용 |
|------|------|
| 1차 오류 | "Agent Skill"을 확인 없이 "자동화 도구"로 해석 |
| 2차 오류 | 지적 직후 "Agent Skill = 단순 지침서"라고 또 단정 |

### 원인

- 모르는 용어를 확인 없이 추측으로 해석
- 빠르게 답변하려는 압박
- "모른다"고 말하기 불편함

### 교훈

**CLAUDE.md에 "답변 전 필수 확인 규칙" 추가됨:**

```markdown
## 답변 전 필수 확인 규칙

**외부 용어/개념/도구에 대해 답변할 때:**

1. 공식 문서나 신뢰할 수 있는 출처를 먼저 확인한다
2. 확인 없이는 해석하지 않는다
3. 확인할 수 없으면 "모른다"고 말한다
```

---

## 3. 올바른 평가 기준

### 잘못된 평가 기준

"얼마나 자동화할 수 있는가?" (X)

### 올바른 평가 기준

"Claude가 이 파이프라인을 참고해서 일관된 품질의 작업을 수행할 수 있는가?" (O)

즉, **"품질 보장 도구"**로서의 평가:
- 누가 작업하든 동일한 프로세스
- 누가 작업하든 동일한 체크리스트
- 누가 작업하든 동일한 품질 기준
- 누가 작업하든 동일한 산출물 구조

---

## 4. 현재 파이프라인 평가

### 평가 항목

| 요소 | Figma_Conversion | RNBT_architecture |
|------|------------------|-------------------|
| 명확한 프로세스 정의 | ✅ | ✅ |
| 단계별 지침 | ✅ (851줄 CLAUDE.md) | ✅ (README.md) |
| 품질 기준 / 체크리스트 | ✅ (스크린샷 검증) | 🔶 (암묵적, 예제 기반) |
| 예외 처리 규칙 | ✅ (gap, box-sizing 등) | 🔶 (작업 원칙만 있음) |
| 참고할 예제 | ✅ (Conversion/) | ✅ (Examples/, Projects/) |
| 재사용 가능한 코드/템플릿 | 🔶 (없음) | ✅ (Utils/, Mixin 패턴) |

### 판정

**🔶 Agent Skill로 변환 가능하나, 일부 보완 필요**

- Figma_Conversion: 지침은 상세하나 재사용 템플릿 부족
- RNBT_architecture: 예제는 풍부하나 체크리스트가 암묵적

---

## 5. 실행 환경 고려사항

### 사전 설정 요구사항 (사용자가 직접 해야 함)

| 항목 | 설명 | Agent Skill로 해결 가능? |
|------|------|------------------------|
| npm install | playwright 등 의존성 설치 | ✅ scripts/로 가능 |
| npx playwright install | 브라우저 설치 | ✅ scripts/로 가능 |
| Figma Desktop 실행 | 앱 실행 | ❌ 사용자가 수동으로 |
| Figma MCP 서버 연결 | claude_desktop_config.json 설정 | ❌ 사용자가 수동으로 |
| Figma 파일 접근 권한 | Figma 계정 및 파일 권한 | ❌ 사용자가 수동으로 |
| 로컬 서버 실행 | npx serve | ✅ scripts/로 가능 |

### 해결 가능 vs 불가능 정리

**Agent Skill로 해결 가능:**
- npm 의존성 설치
- Playwright 브라우저 설치
- 로컬 서버 실행
- 스크린샷 캡처/비교
- 파일/폴더 생성

**사용자가 직접 설정해야 함:**
- Figma Desktop 실행
- Figma MCP 서버 연결 (claude_desktop_config.json)
- Figma 파일 접근 권한

→ **사전 설정 가이드 문서 필요**

---

## 6. 다음 단계

### Agent Skill 패키징 방향

```
.claude/skills/figma-to-rnbt/
├── SKILL.md                 ← 전체 파이프라인 지침
├── figma-conversion.md      ← Figma → HTML/CSS 상세 가이드
├── rnbt-component.md        ← HTML/CSS → 동적 컴포넌트 가이드
├── prerequisites.md         ← 사전 설정 가이드 (Figma MCP 등)
├── scripts/
│   ├── setup.js             ← npm install, playwright install (Node.js)
│   └── screenshot.js        ← Playwright 스크린샷 캡처
└── templates/
    ├── component-template/  ← 컴포넌트 폴더 구조 템플릿
    └── register-template.js ← register.js 보일러플레이트
```

### 보완이 필요한 항목

| 항목 | 현재 상태 | 필요한 작업 |
|------|----------|------------|
| 사전 설정 가이드 | 없음 | prerequisites.md 작성 |
| RNBT 체크리스트 | 암묵적 | 명시적 체크리스트 문서화 |
| 컴포넌트 템플릿 | 없음 | templates/ 폴더 구성 |
| 스크립트 | 없음 | setup.js, screenshot.js 작성 |

---

## 7. 프로젝트 효용 평가

### 이 프로젝트가 해결하는 문제

| 문제 | 해결책 |
|------|--------|
| Figma → 코드 변환 시 수동 작업 많음 | Figma MCP로 정확한 데이터 추출 |
| 변환 품질 불일치 | 스크린샷 검증으로 시각적 정확성 보장 |
| 반복적인 프로세스 | 상세한 지침서로 일관된 프로세스 |
| 웹 빌더 특화 출력 필요 | RENOBIT 컨테이너 구조 규칙 적용 |

### 효용의 범위

| 대상 | 효용 |
|------|------|
| RENOBIT 웹 빌더 사용자 | ✅ 높음 (특화된 출력 형식) |
| Figma MCP 환경 사용자 | ✅ 높음 (파이프라인 활용 가능) |
| Claude Code 사용자 | ✅ 있음 (Agent Skill로 활용) |
| 일반 개발자 (다른 웹 빌더) | ⚠️ 제한적 (RENOBIT 특화) |

### 진입 장벽

1. Figma Desktop + MCP 서버 설정 필요
2. RENOBIT 웹 빌더 이해 필요
3. 초기 npm 설정 필요

→ **사전 설정 가이드로 진입 장벽 낮출 수 있음**

---

## 8. 결론

### 현재 상태

1. **Agent Skill로 변환 가능** (단, 일부 보완 필요)
   - 상세한 지침 문서 존재
   - 명확한 프로세스 정의
   - 풍부한 예제

2. **보완 필요 항목**
   - 사전 설정 가이드 (prerequisites.md)
   - RNBT 체크리스트 명시화
   - 재사용 템플릿 구성
   - 실행 스크립트 작성

3. **프로젝트 효용**
   - RENOBIT 웹 빌더 + Figma MCP 환경에서 명확한 효용
   - 범용 도구는 아님 (특화된 도구)
   - 사전 설정이 진입 장벽이나 가이드로 해결 가능

### 다음 액션

1. 사전 설정 가이드 작성
2. RNBT 체크리스트 문서화
3. Agent Skill 구조로 패키징
4. 테스트 및 검증

---

*최종 업데이트: 2025-12-26*
