# Unity Development 플러그인

[![Version](https://img.shields.io/badge/Version-0.2.0-green.svg)]()

Unity + C# 환경에서 **분석 → 기획 → 설계 → 구현**의 전체 개발 사이클을 AI와 함께 체계적으로 진행할 수 있는 스킬, 커맨드, 개발 가이드라인을 제공합니다.

---

## 스킬 (Skills)

| 스킬 | 설명 |
|------|------|
| [`game-analysis`](#game-analysis) | 참조 게임 MDA 분석 보고서 생성 |
| [`game-prd`](#game-prd) | 분석 보고서 기반 PRD(제품 요구사항 명세서) 작성 |
| [`game-feature`](#game-feature) | PRD / FEATURES.md 기반 기능 목록 관리 |
| [`unity-env-analyzer`](#unity-env-analyzer) | Unity 프로젝트 분석 → `UNITY_DEV_ENV.md` 자동 생성 |
| [`unity-conventions-analyzer`](#unity-conventions-analyzer) | 코드 아키텍처·컨벤션 분석 → `UNITY_PROJECT_CONVENTIONS.md` 자동 생성 |
| [`unity-plan`](#unity-plan) | 기능 요청 → 작업 체크리스트 분해 |
| [`unity-tdd`](#unity-tdd) | TDD(Red → Green → Refactor) 사이클 기반 구현 |
| [`unity-uitoolkit`](#unity-uitoolkit) | Unity UI Toolkit(UXML + USS + C#) 기반 UI 구현 가이드 |
| [`project-bugfix`](#project-bugfix) | 버그 원인 파악 · 수정 · 안전 장치 추가 |
| [`project-commit`](#project-commit) | 변경 내용 분석 → 일관된 형식의 커밋 메시지 작성 및 커밋 |

---

#### `game-analysis`
참조 게임을 웹 검색으로 분석하고 MDA 프레임워크 기반의 표준화된 게임 분석 보고서를 생성합니다.

- MDA 프레임워크 (Mechanics / Dynamics / Aesthetics) 기반 분석
- 장르별 모듈 지원 (Match-3 퍼즐, 하이브리드 캐주얼 타이쿤, 뱀서라이크)
- 플레이어 경험 목표, 게임 루프, 수익화 모델 포함
- `Docs/{게임명}_analysis.md` 자동 생성

#### `game-prd`
게임 분석 보고서를 기반으로 구현 가능한 수준의 PRD(제품 요구사항 명세서)를 작성합니다.

- 기술 스택 독립적 (엔진·언어·프레임워크 언급 없음)
- Feature 단위: 로직 / 비주얼 / 연출 레이어 분리
- 스프린트 플랜 및 의존성 표기 포함
- `Docs/{게임명}_PRD.md` 자동 생성

#### `game-feature`
게임 PRD 또는 FEATURES.md의 기능 목록을 관리합니다.

- PRD 파일 우선 탐색, 없으면 FEATURES.md 사용
- 기능 추가/제거/수정/완료 처리 지원
- 의존성 기반 삽입 위치 자동 결정
- 서브 ID 처리로 기존 ID 체계 보존

#### `unity-env-analyzer`
Unity 프로젝트를 분석하여 개발 환경 문서를 자동 생성합니다.

- 패키지·라이브러리, 폴더 구조, Unity 버전 등 추출
- `UNITY_DEV_ENV.md` 자동 생성

#### `unity-conventions-analyzer`
Unity 프로젝트의 코드베이스를 분석하여 아키텍처와 코딩 컨벤션을 문서화합니다.

- 코드 아키텍처, 디자인 패턴, 네이밍 규칙 역추론
- `UNITY_PROJECT_CONVENTIONS.md` 자동 생성

#### `unity-plan`
기능 요청을 작업 체크리스트로 분해합니다.

- UNITY_DEV_ENV.md 기반 프로젝트 환경 반영
- 의미 있는 작은 단위로 태스크 분해
- 30분~2시간 단위의 작업 분할
- `Docs/Features/{FeatureName}.md` 자동 생성

#### `unity-tdd`
켄트 벡의 TDD와 Tidy First 원칙을 따라 기능을 구현합니다.

- 엄격한 **Red → Green → Refactor** 사이클 준수
- Vertical Slice 방식 (테스트 하나 → 구현 하나)
- EditMode 테스트 우선, PlayMode는 Unity 런타임 필요 시에만
- 구조적 변경과 동작적 변경 분리 (Tidy First)

#### `unity-uitoolkit`
Unity 프로젝트에서 UI를 만들거나 수정할 때 UI Toolkit(UXML + USS + C#)을 사용하도록 안내합니다.

- Canvas/GameObject 기반 uGUI 사용 금지, UIDocument + UXML 구조 강제
- UXML 작성 규칙, USS 스타일링, C# View 패턴 안내
- MVP + VContainer + R3 연동 패턴 제공
- 팝업·동적 UI, 에디터 UI 패턴 포함

#### `project-bugfix`
버그의 근본 원인을 가설 기반으로 탐색하고 수정합니다.

- 가설 → 검증 → 수정 → 안전 장치 흐름
- 코드 문제: 회귀 테스트 자동 추가
- 비코드 문제: 재발 가능성 판단 후 작업 절차 안내
- 수정 실패 시 변경사항 롤백 후 재분석

#### `project-commit`
저장소의 변경 내용을 분석하여 일관된 형식의 커밋 메시지를 작성하고 커밋합니다.

- 타입 자동 분류: `feat` / `fix` / `balance` / `ui` / `perf` / `refactor` / `chore`
- 스토어 업데이트 노트 추출에 최적화된 형식
- 커밋 전 메시지 초안을 사용자에게 확인 후 실행
- Tidy First 원칙에 따라 혼합 커밋 분리 권장

---

## 커맨드 (Commands)

스킬을 조합해 복잡한 워크플로우를 한 번에 실행합니다.

#### `/blueprint {게임명 또는 식별 정보}`
게임 분석 보고서와 PRD를 자동으로 순차 생성합니다.

```
/blueprint Candy Crush Saga
/blueprint https://play.google.com/store/apps/details?id=com.king.candycrushsaga
/blueprint 퍼즐 게임, 화살표를 탭해서 탈출시키는 모바일 게임
```

- `game-analysis` → `game-prd` 순서로 스킬 자동 실행
- 출력: `Docs/{게임명}_analysis.md` + `Docs/{게임명}_PRD.md`

#### `/implement [{기능 설명}]`
기능 목록의 다음 항목을 가져와 세부 계획을 수립하고, TDD 사이클에 따라 개발·테스트·커밋·머지까지 완료합니다.

```
/implement                          # 기능 목록의 다음 항목 구현
/implement 하트 자동 회복 기능 추가  # 기능 목록에 추가 후 구현
```

- PRD 파일 우선, 없으면 FEATURES.md에서 기능 목록 탐색
- feature 브랜치 생성 → `unity-plan`으로 작업 명세서 생성 → 사용자 승인
- `unity-tdd` 사이클로 구현, 태스크마다 자동 커밋
- 전체 테스트 통과 후 `project-commit`으로 최종 커밋 메시지 작성
- squash merge로 단일 커밋으로 원래 브랜치에 머지

---

## 개발 규칙 (Rules)

Claude Code가 코드 생성 및 설계 시 준수하는 가이드라인입니다.

### 코드 스타일 (`code-style.md`)
- PascalCase 클래스, camelCase 변수, `_prefix` 프라이빗 필드
- Early return으로 중첩 최소화
- UniTask (코루틴 대체), Awake에서 컴포넌트 캐싱
- XML 문서 주석 사용 금지

---

## 사용 가이드

### A. 신규 프로젝트 — 처음부터 시작하기

참조 게임을 분석하고 PRD를 자동 생성한 뒤, 기능을 하나씩 구현해 나갑니다.

```
1. Unity 프로젝트 생성 후 플러그인 설치

2. 프로젝트 환경 분석
   → "프로젝트 분석해줘"              # unity-env-analyzer → UNITY_DEV_ENV.md 생성

3. 게임 기획
   → /blueprint Candy Crush Saga     # 분석 보고서 + PRD 자동 생성

4. 기능 구현 (PRD의 기능 목록을 순서대로 처리)
   → /implement                      # 다음 기능 자동 선택 → 계획 → TDD 구현 → 커밋 → 머지
   → /implement                      # 반복...

5. 버그 발생 시
   → "이 버그 수정해줘: {증상}"       # project-bugfix 자동 발동
```

### B. 기존 프로젝트 — 이미 진행 중인 프로젝트에 도입하기

기존 코드베이스의 환경과 컨벤션을 먼저 분석한 뒤, 플러그인의 스킬을 활용합니다.

```
1. 플러그인 설치

2. 프로젝트 환경 분석
   → "프로젝트 분석해줘"              # unity-env-analyzer → UNITY_DEV_ENV.md 생성

3. 코드 컨벤션 분석
   → "코드 컨벤션 분석해줘"           # unity-conventions-analyzer → UNITY_PROJECT_CONVENTIONS.md 생성

4. 기능 목록 준비 (아래 중 택 1)
   a) PRD가 있는 경우: 기능 목록 섹션이 있는 *prd*.md 파일을 프로젝트에 배치
   b) PRD가 없는 경우: "기능 추가해줘: {기능 설명}" → FEATURES.md 자동 생성

5. 기능 구현
   → /implement                      # 기능 목록에서 다음 항목 자동 선택 → 구현
   → /implement {새 기능 설명}        # 기능 목록에 추가 후 바로 구현

6. 개별 스킬 활용 (필요 시)
   → "이 기능 작업 분해해줘"           # unity-plan → 작업 명세서 생성
   → "커밋해줘"                       # project-commit → 표준 커밋
   → "이 버그 수정해줘"               # project-bugfix → 가설 기반 수정
```

### 주요 차이점

| | 신규 프로젝트 | 기존 프로젝트 |
|---|---|---|
| **시작점** | `/blueprint`로 기획부터 | 환경·컨벤션 분석부터 |
| **기능 목록** | PRD 자동 생성 | 기존 PRD 활용 또는 FEATURES.md 생성 |
| **컨벤션 분석** | 코드가 없으므로 생략 | `unity-conventions-analyzer`로 기존 스타일 파악 |

---

## 권장 사용 흐름

```
[프로젝트 초기 설정]
1. unity-env-analyzer         → UNITY_DEV_ENV.md 생성
2. unity-conventions-analyzer  → UNITY_PROJECT_CONVENTIONS.md 생성 (기존 프로젝트)

[게임 기획]
3. /blueprint {게임명}         → 분석 보고서 + PRD 자동 생성
                                 (game-analysis → game-prd 순차 실행)

[기능 개발]
4. /implement                  → 다음 기능 계획 수립 후 TDD로 구현·커밋·머지
```

---

## 플러그인 구조

```
unity-development/
├── .claude-plugin/
│   └── plugin.json                  # 플러그인 매니페스트
├── skills/
│   ├── game-analysis/               # 게임 분석 보고서 생성
│   ├── game-prd/                    # PRD(제품 요구사항 명세서) 생성
│   ├── game-feature/                # 기능 목록 관리
│   ├── unity-env-analyzer/          # Unity 개발 환경 분석
│   ├── unity-conventions-analyzer/  # 코드 컨벤션 분석
│   ├── unity-plan/                  # 기능 작업 체크리스트 생성
│   ├── unity-tdd/                   # TDD 사이클 기반 구현
│   ├── unity-uitoolkit/             # Unity UI Toolkit 기반 UI 구현 가이드
│   ├── project-bugfix/              # 버그 원인 파악 및 수정
│   └── project-commit/              # 커밋 메시지 표준화 및 커밋
├── rules/
│   └── code-style.md                # C#/Unity 코드 스타일
└── commands/
    ├── blueprint.md                 # 분석 보고서 + PRD 자동 생성
    └── implement.md                 # 기능 구현 (계획~머지 일괄)
```

---

## 기술 스택

플러그인이 대상으로 하는 Unity 게임 개발 환경:

| 분류 | 라이브러리 |
|------|-----------|
| **DI** | VContainer (또는 Zenject, Pure DI) |
| **이벤트** | MessagePipe, UniRx, R3 |
| **비동기** | UniTask |
| **트위닝** | DOTween, LeanTween |
| **테스트** | NUnit, NSubstitute, Moq |
| **UI** | UI Toolkit, MonoBehaviour 기반 MVP |
