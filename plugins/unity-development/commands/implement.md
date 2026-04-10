---
name: implement
description: 기능 목록에서 다음 개발 항목을 가져와 세부 계획 수립, TDD 구현, 커밋, 머지까지 일괄 진행하는 커맨드. "/implement", "/implement {기능 설명}", "다음 기능 구현해줘" 등의 요청 시 사용.
argument-hint: [기능 설명]
---

# /implement 커맨드

기능 목록에서 다음 개발 항목을 가져와 세부 계획을 수립하고, 계획에 따라 개발·테스트까지 완료한다.
추가 내용이 있는 경우 기능 목록을 먼저 업데이트한 뒤 동일하게 진행한다.

---

## 사용법

```
/implement              # 기능 목록의 다음 항목 구현
/implement {기능 설명}  # 해당 내용을 기능 목록에 추가/수정 후 구현
```

**예시**:
```
/implement
/implement 하트 자동 회복 기능 추가
/implement F-003 설명 수정 후 구현
```

---

## 실행 절차

### Step 1: 기능 목록 파일 탐색

기능 목록을 관리하는 파일을 다음 우선순위로 탐색한다.

1. **PRD 파일**: 파일명에 `prd`가 포함된 `.md` 파일 (`find . -maxdepth 3 -type f -iname "*prd*.md"`)
2. **FEATURES.md**: 프로젝트 루트의 `FEATURES.md`

**파일을 찾은 경우**: 파일을 읽고 기능 목록 섹션에서 미완료 항목(`- [ ]`)을 파악한다.
**파일을 찾지 못한 경우**: "기능 목록 파일이 없습니다. 개발할 기능을 입력해 주세요." 안내 후 종료.

---

### Step 2: 작업 환경 준비

구현을 시작하기 전에 버전 관리 중인 파일의 변경 사항을 확인한다.

```bash
git status --short --untracked-files=no
```

**변경 사항이 있는 경우**: 사용자에게 정리를 요청하고 대기한다.

```
⚠️ 저장소에 커밋되지 않은 변경 사항이 있습니다.
   {변경 파일 목록}

구현을 시작하기 전에 변경 사항을 커밋하거나 스태시해 주세요.
정리가 완료되면 "완료" 또는 "continue"를 입력해 주세요.
```

**변경 사항이 없는 경우**: 기능 브랜치를 생성하고 전환한다. 버전 관리가 안 되는 파일(untracked)은 무시한다.

```bash
git checkout -b feature/{feature-name}
```

- 브랜치명은 `feature/{feature-name}` 형식 (영문, kebab-case)
- 기능명에서 핵심 키워드를 추출하여 브랜치명으로 사용한다
  - 예: "하트 자동 회복 기능" → `feature/heart-auto-recovery`

```
🌿 브랜치 생성: feature/{feature-name}
```

---

### Step 3: 입력 분기

`$ARGUMENTS` 값에 따라 분기한다.

**A. 인자 없음**
- Step 1에서 파악한 미완료 항목 중 첫 번째(다음 개발 항목)를 선택한다.
- 미완료 항목이 없으면: "기능 목록에 미완료 항목이 없습니다. 개발할 기능을 입력해 주세요." 안내 후 종료.
- 항목이 있으면 바로 Step 4로 진행.

**B. 인자 있음**
- `game-feature` 스킬의 워크플로우를 실행한다.
  - 스킬 파일: `.claude/skills/game-feature/SKILL.md` — **반드시 먼저 읽고** 해당 스킬 절차를 따른다.
- 기능 목록 업데이트 완료 후 Step 4로 진행.

---

### Step 4: 구현 대상 확정

Step 3에서 결정된 기능을 사용자에게 알린다:

```
▶ 구현 대상: [{ID}] {기능명}
   {기능 설명}

세부 계획을 수립합니다...
```

---

### Step 5: 세부 계획 수립 (unity-plan 스킬 실행)

`unity-plan` 스킬의 워크플로우 전체를 실행한다.

- 스킬 파일: `.claude/skills/unity-plan/SKILL.md` — **반드시 먼저 읽고** 해당 스킬의 워크플로우를 그대로 따른다.
- 인풋: Step 4에서 확정된 기능명 및 설명
- 출력: `Docs/Features/{FeatureName}.md` 작업 명세서

unity-plan 스킬은 계획 수립 후 사용자 승인을 받는다. 승인 없이 Step 6으로 넘어가지 않는다.

---

### Step 6: 계획에 따른 개발 실행

Step 5에서 생성된 작업 명세서의 체크리스트를 **순서대로** 실행한다.

**실행 원칙**:
- 코드 작업은 `unity-tdd` 스킬의 워크플로우를 따른다.
  - 스킬 파일: `.claude/skills/unity-tdd/SKILL.md` — **반드시 먼저 읽고** 해당 스킬의 TDD 사이클(Red → Green → Refactor)을 적용한다.
- 각 태스크 완료 시 작업 명세서의 해당 체크박스를 `[x]`로 업데이트한다.
- **각 태스크 완료 시마다** feature 브랜치에 자동 커밋한다. 사용자 확인 없이 즉시 커밋한다.
  - 커밋 메시지: `wip: {태스크 요약}`
  - 이 커밋들은 Step 8에서 squash merge로 하나의 커밋으로 합쳐진다.
- 수동 작업 항목이 있으면 사용자에게 안내하고 완료 확인을 요청한 뒤 다음 단계로 진행한다.

**수동 작업 처리**:
```
⚠️ 수동 작업 필요:
   {수동 작업 내용}

완료되면 "완료" 또는 "continue"를 입력해 주세요.
```

---

### Step 7: 테스트 실행 및 확인

모든 태스크 완료 후 전체 테스트를 실행한다.

- **EditMode 테스트**: 먼저 실행 (PlayMode 테스트 제외)
- 테스트 실패 시: 실패 원인을 분석하고 수정한 뒤 다시 실행한다. 반복 실패 시 사용자에게 상황을 보고한다.
- 모든 테스트 통과 확인 후 Step 8로 진행.

```
✅ 테스트 통과
```

---

### Step 8: 기능 목록 완료 표시

구현된 기능을 기능 목록 파일에 완료 표시한다.

- `game-feature` 스킬의 "완료 표시" 절차를 따른다.
- 해당 항목: `[ ]` → `[x]`, 날짜 갱신.

---

### Step 9: 커밋 및 머지

`project-commit` 스킬을 실행하여 최종 커밋 메시지를 작성한다.

- 스킬 파일: `.claude/skills/project-commit/SKILL.md` — **반드시 먼저 읽고** 해당 스킬 절차를 따른다.
- 이 단계에서는 사용자 확인을 받는다.

사용자가 커밋 메시지를 승인하면, feature 브랜치의 모든 커밋을 squash merge로 단일 커밋으로 합쳐 원래 브랜치에 머지한다.

```bash
git checkout {원래 브랜치}
git merge --squash feature/{feature-name}
git commit -m "{project-commit 스킬이 작성한 커밋 메시지}"
```

머지 완료 후 기능 브랜치를 삭제한다.

```bash
git branch -d feature/{feature-name}
```

```
🔀 머지 완료: feature/{feature-name} → {원래 브랜치} (squash)
🗑️ 브랜치 삭제: feature/{feature-name}
```

---

### Step 10: 완료 요약 출력

```
✅ /implement 완료

🎯 구현 기능: [{ID}] {기능명}
📋 작업 명세서: Docs/Features/{FeatureName}.md
🔀 머지: feature/{feature-name} → {원래 브랜치}
```

---

## 주의사항

- 수동 작업이 필요한 경우 건너뛰지 않고 반드시 사용자에게 안내한다.
- 각 스킬의 품질 기준을 충족한 후 다음 단계로 넘어간다.
- 코드 작업 시 unity-tdd 스킬의 TDD 사이클을 반드시 준수한다.
