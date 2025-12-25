# Git Submodule 가이드

## 서브모듈이란?

독립적인 Git 저장소를 다른 저장소 안에 포함시키는 방식입니다.

```
FIGMA_INTEGRATED/          ← 부모 저장소
├── Figma_Conversion/      ← 서브모듈 (독립 저장소)
├── RNBT_architecture/     ← 서브모듈 (독립 저장소)
```

---

## 핵심 개념

**부모 저장소는 서브모듈의 "커밋 해시"를 저장합니다. 브랜치 이름이 아닙니다.**

```
부모가 기억하는 것: Figma_Conversion → abc123 (커밋 해시)
```

따라서 `git submodule update` 하면 항상 **detached HEAD** 상태가 됩니다.

```bash
cd Figma_Conversion
git status
# HEAD detached at abc123   ← 브랜치가 아닌 커밋 해시
```

---

## 자주 쓰는 명령어

### 서브모듈 최신화

```bash
# 부모가 기억하는 커밋으로 체크아웃 (안전)
git submodule update --recursive

# 원격 최신 커밋으로 업데이트 (부모와 불일치 발생 가능)
git submodule update --remote --recursive
```

### 서브모듈에서 브랜치 작업

```bash
cd Figma_Conversion
git checkout main          # 브랜치로 이동
# 작업...
git add . && git commit && git push

cd ..
git add Figma_Conversion   # 부모가 기억하는 커밋 갱신
git commit -m "chore: 서브모듈 업데이트"
git push
```

### 클론 시 서브모듈 포함

```bash
git clone --recursive <저장소 URL>

# 또는 클론 후
git submodule update --init --recursive
```

---

## 자주 발생하는 상황

### 1. 다른 로컬에서 작업 후 돌아왔더니 브랜치가 커밋 해시로 변경됨

**원인**: `git submodule update`가 실행되면 항상 detached HEAD가 됩니다.

**해결**: 서브모듈 내에서 브랜치로 이동
```bash
cd Figma_Conversion
git checkout main
```

### 2. `modified: Figma_Conversion (new commits)` 표시

**원인**: 서브모듈의 현재 커밋과 부모가 기억하는 커밋이 다름

**해결 A**: 부모가 기억하는 커밋으로 되돌리기
```bash
git submodule update --recursive
```

**해결 B**: 부모가 새 커밋을 기억하도록 갱신
```bash
git add Figma_Conversion
git commit -m "chore: 서브모듈 업데이트"
```

---

## 서브모듈 vs 일반 폴더

| 항목 | 서브모듈 | 일반 폴더 |
|------|----------|-----------|
| 독립적 버전 관리 | O | X |
| 다른 프로젝트에서 재사용 | 쉬움 | 복사 필요 |
| 동기화 | 번거로움 | 자동 |
| 명령어 복잡도 | 높음 | 낮음 |

**권장**: 진짜 독립적으로 관리해야 할 때만 서브모듈 사용

---

*최종 업데이트: 2025-12-26*
