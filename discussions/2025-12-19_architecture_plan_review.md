# 아키텍처 계획 검토 의견서

**작성일**: 2025-12-19
**검토자**: Claude (시니어 개발자 관점)

---

## 1. 전체 계획 요약

### 기반 컴포넌트

| 레이어 | 기반 컴포넌트 | Mixin |
|--------|---------------|-------|
| 2D | FreeCode | FreeCodeMixin |
| 3D | ModelLoaderComponent | ModelLoaderMixin |

### 2D 컴포넌트 파이프라인

```
Figma → Figma_Conversion (정적) → preview.html 검증 → RNBT (동적)
```

### 컴포넌트 유형

| 유형 | 특징 | 예시 |
|------|------|------|
| 일반 (수동적) | 페이지가 오케스트레이션 | 대부분의 2D 컴포넌트 |
| Functional (자기완결) | 자체 fetch, 이벤트 전파 → 메소드 실행 | TemperatureSensor (3D) |

---

## 2. 계획의 강점 (Well-Designed)

### ✅ 명확한 단계 분리
- 정적(퍼블리싱) vs 동적(스크립트) 책임 분리
- Figma_Conversion은 시각적 정확성만 책임
- RNBT_architecture는 동작만 책임
- → 각 단계의 검증이 명확함

### ✅ Mixin 기반 확장성
- 기본 컴포넌트(FreeCode, ModelLoader)는 최소한만 정의
- Mixin으로 기능 주입 (팝업, 차트, 데이터 로딩 등)
- → 조합으로 다양한 컴포넌트 생성 가능

### ✅ Config 패턴으로 추상화
- TimeTrendChart의 chartConfig 예시
- 메소드는 그대로, 데이터 매핑만 변경
- → 동일 메소드의 재사용성 극대화

### ✅ 자기완결 vs 수동적 구분
- Functional: 복잡한 도메인 로직을 캡슐화
- 수동적: 페이지가 제어, 컴포넌트는 뷰만 담당
- → 적재적소에 맞는 패턴 선택 가능

### ✅ Shadow DOM 활용
- 팝업 스타일 격리
- 기존 페이지 스타일과 충돌 방지
- → 안정적인 UI 렌더링

### ✅ 마스터 레이어 활성화
- `reload_master_on_page_change` 옵션 추가 완료
- 공통 영역 스크립트 실행 가능
- → 네비게이션, 헤더 등 공통 요소 관리 용이

---

## 3. 현재 준비 상태 평가

| 항목 | 상태 | 비고 |
|------|------|------|
| ComponentMixin.js (front) | ✅ 완료 | FreeCode, ModelLoader Mixin |
| ComponentMixin.js (RNBT) | ✅ 완료 | 참조 구현 |
| Mixin.js (Shadow DOM 팝업) | ✅ 완료 | createPopup, createChart |
| WKit.js (유틸리티) | ✅ 완료 | 바인딩, 3D, 정리 |
| GlobalDataPublisher | ✅ 완료 | Topic Pub-Sub |
| WEventBus | ✅ 완료 | 컴포넌트 간 통신 |
| fx.js | ✅ 완료 | 함수형 유틸 |
| IPSILON_3D 예제 | ✅ 완료 | 자기완결 컴포넌트 데모 |
| API 명세 (API_SPEC.md) | ✅ 완료 | 배치 검증 API 포함 |
| Mock Server | ✅ 완료 | Express 기반 |
| Asset 검증 (OpenPageCommand) | ✅ 완료 | Editor 삭제, Viewer 경고 |
| 마스터 재로드 옵션 | ✅ 완료 | reload_master_on_page_change |
| 컴포넌트 템플릿 생성기 | ❌ 미완 | boilerplate 자동화 필요 |
| Config 패턴 표준 문서 | ❌ 미완 | chartConfig 외 패턴 정리 |
| 3D 자기완결 공통 패턴 | 🔶 부분 | TemperatureSensor만 존재 |
| E2E 테스트 자동화 | ❌ 미완 | 스크린샷 비교 자동화 |

**준비 완성도: 약 75%**

---

## 4. 대량 생산을 위한 제안

### 제안 1: 컴포넌트 스캐폴딩 CLI

**현재**: 수동으로 폴더/파일 생성
**제안**: CLI 도구로 자동 생성

```bash
$ rnbt create component TemperatureSensor --type=functional --layer=3d
```

**생성 결과**:
```
TemperatureSensor/
├── TemperatureSensor.html
├── TemperatureSensor.css
├── register.js          # Mixin 적용, 이벤트 바인딩 템플릿
├── destroy.js           # 정리 템플릿
└── tb_package.json      # 메타데이터
```

**효과**: 반복 작업 제거, 표준 구조 강제

---

### 제안 2: Config 패턴 카탈로그

**현재**: chartConfig만 문서화
**제안**: 모든 Config 패턴 카탈로그화

| Config | 용도 |
|--------|------|
| chartConfig | ECharts 옵션 매핑 |
| datasetConfig | API 엔드포인트 매핑 |
| bindingConfig | DOM 이벤트 → Topic 매핑 |
| renderConfig | 데이터 → DOM 렌더링 매핑 |

**효과**: 새 컴포넌트 작성 시 "어떤 Config가 필요한가?" 명확

---

### 제안 3: 3D 자기완결 컴포넌트 베이스 클래스

**현재**: TemperatureSensor가 유일한 예시
**제안**: 공통 패턴 추출하여 Mixin 또는 베이스로 분리

**Functional3DMixin 제공 기능**:
- `showPopup()` / `hidePopup()`
- `fetchData()` 래퍼
- `datasetInfo` 구조 표준화
- 이벤트 발행 헬퍼 (`@xxxClicked`)

**효과**: 3D 자기완결 컴포넌트 작성 시간 단축

---

## 5. 잠재적 리스크 및 고려사항

### ⚠️ 리스크 1: front와 RNBT 동기화

- front의 ComponentMixin과 RNBT의 ComponentMixin이 분리되어 있음
- 한쪽을 수정하면 다른 쪽도 수정해야 함
- **제안**: 단일 소스로 통합하거나, 버전 태깅 후 복사 스크립트

### ⚠️ 리스크 2: Figma 변경 시 재작업

- Figma 디자인이 변경되면 Conversion부터 다시 시작
- 이미 스크립트가 입혀진 컴포넌트는?
- **제안**: HTML/CSS만 교체 가능한 구조 유지 (스크립트 분리 철저)

### ⚠️ 리스크 3: 자기완결 컴포넌트의 테스트

- 내부에 fetch가 있어 단위 테스트 어려움
- **제안**: fetch 함수를 주입 가능하게 설계 (의존성 주입)

---

## 6. 최종 의견

### 결론

계획은 **매우 체계적이고 실현 가능**합니다.

### 강점
- 이미 핵심 인프라(Mixin, Utils, 예제)가 갖춰져 있음
- 단계별 분리가 명확하여 검증과 디버깅이 용이
- Config 패턴으로 추상화 방향이 올바름

### 보완 필요
- 대량 생산을 위한 자동화 도구 (스캐폴딩)
- 3D Functional 컴포넌트의 공통 패턴 추출
- front ↔ RNBT 동기화 전략

### 다음 단계 추천

1. **IPSILON_3D에서 2~3개 더 3D 자기완결 컴포넌트 만들어보기**
   - 공통 패턴 발견 → Functional3DMixin 추출

2. **Config 패턴 카탈로그 문서화**
   - 새 컴포넌트 작성 가이드로 활용

3. **스캐폴딩 스크립트 (간단한 Node.js)**
   - 폴더/파일 자동 생성

**현재 75%의 준비도에서, 위 3가지만 추가하면 90% 이상 준비 완료**

---

*최종 업데이트: 2025-12-19*
