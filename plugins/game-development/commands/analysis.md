# /analysis 커맨드

게임 이름, URL, 또는 게임을 특정할 수 있는 정보를 입력받아
**게임 분석 보고서 → PRD** 를 자동으로 순차 생성한다.

---

## 사용법

```
/analysis {게임명 또는 식별 정보}
```

**예시**:
```
/analysis Candy Crush Saga
/analysis https://play.google.com/store/apps/details?id=com.king.candycrushsaga
/analysis 퍼즐 게임, 화살표를 탭해서 탈출시키는 모바일 게임
```

---

## 실행 절차

### Step 1: 입력 확인

`$ARGUMENTS`에서 게임 정보를 추출한다.

- 값이 비어 있으면: "분석할 게임 이름 또는 URL을 입력해 주세요." 안내 후 종료.
- 값이 있으면: 게임명을 확정하고 Step 2로 진행.

URL이 제공된 경우 URL에서 게임명을 추출하거나 URL을 검색 단서로 활용한다.
설명만 제공된 경우 설명에서 가장 가까운 실제 게임명을 추론한다.

---

### Step 2: 게임 분석 보고서 생성 (dk-analysis 스킬 실행)

`dk-analysis` 스킬의 워크플로우 전체를 실행한다.

- 스킬 파일: `.claude/skills/dk-analysis/SKILL.md`
- **반드시 이 파일을 먼저 읽고** 해당 스킬의 워크플로우(Step 1~4)를 그대로 따른다.
- 출력: `/mnt/user-data/outputs/{게임명}_analysis.md`

완료 후 사용자에게 진행 상황을 알린다:
```
✅ 분석 보고서 생성 완료: {게임명}_analysis.md
   → PRD 생성을 시작합니다...
```

---

### Step 3: PRD 생성 (dk-grd 스킬 실행)

Step 2에서 생성된 분석 보고서를 인풋으로 `dk-grd` 스킬을 실행한다.

- 스킬 파일: `.claude/skills/dk-grd/SKILL.md`
- **반드시 이 파일을 먼저 읽고** 해당 스킬의 워크플로우(Step 1~4)를 그대로 따른다.
- 인풋: Step 2에서 생성한 `{게임명}_analysis.md` 파일 (파일 경로로 직접 전달)
- 출력: `/mnt/user-data/outputs/{게임명}_PRD.md`

---

### Step 4: 완료 요약 출력

두 파일이 모두 생성되면 아래 형식으로 결과를 출력한다:

```
✅ /analysis 완료

📊 분석 보고서: {게임명}_analysis.md
📋 PRD:         {게임명}_PRD.md

두 파일을 present_files로 전달합니다.
```

`present_files`로 두 파일을 사용자에게 전달한다.

---

## 주의사항

- dk-analysis → dk-grd 순서는 반드시 지킨다. PRD는 분석 보고서 없이 생성할 수 없다.
- 각 스킬의 품질 기준을 모두 충족한 후 다음 단계로 넘어간다.
- 분석 보고서의 정보가 부족한 경우에도 추정값을 포함해 PRD를 완성한다.
