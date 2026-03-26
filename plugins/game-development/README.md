# Game Development 플러그인

[![Version](https://img.shields.io/badge/Version-0.0.6-green.svg)]()

Unity + C# 환경에서 **분석 → 기획 → 설계 → 구현**의 전체 개발 사이클을 AI와 함께 체계적으로 진행할 수 있는 스킬, 커맨드, 개발 가이드라인을 제공합니다.

---

## 설치

```bash
/plugin install game-development
```

---

## 스킬 (Skills)

| 스킬 | 설명 |
|------|------|
| [`/project-context`](#project-context) | Unity 프로젝트 분석 → `project-context.md` 자동 생성 |
| [`/dk-analysis`](#dk-analysis) | 참조 게임 MDA 분석 보고서 생성 |
| [`/dk-grd`](#dk-grd) | 분석 보고서 기반 GRD(게임 요구사항 명세서) 작성 |
| [`/dk-plan`](#dk-plan) | 기능 요청 → TDD 작업 체크리스트 분해 |
| [`/dk-feature`](#dk-feature) | `features.md` 기반 개발 기능 목록 관리 |
| [`/dk-bugfix`](#dk-bugfix) | 버그 원인 파악 · 수정 · 회귀 테스트 추가 |

---

#### `/project-context`
기존 Unity 프로젝트를 분석하여 Claude Code가 참조할 수 있는 컨텍스트 문서를 자동 생성합니다.

- 패키지·라이브러리, 폴더 구조, 코딩 스타일, 핵심 패턴, 테스트 구조 추출
- `project-context.md` 자동 생성 (docs 폴더 우선, 없으면 `./docs/` 생성)

#### `/dk-analysis`
참조 게임을 웹 검색으로 분석하고 MDA 프레임워크 기반의 표준화된 게임 분석 보고서를 생성합니다.

- MDA 프레임워크 (Mechanics / Dynamics / Aesthetics) 기반 분석
- 플레이어 경험 목표, 게임 루프, 수익화 모델 포함
- `{게임명}_analysis.md` 자동 생성

#### `/dk-grd`
게임 분석 보고서를 기반으로 구현 가능한 수준의 GRD(게임 요구사항 명세서)를 작성합니다.

- 핵심 게임플레이 루프, 시스템 명세, UX/UI 스펙 포함
- MVP 범위 및 스프린트 계획 포함
- `{게임명}_GRD.md` 자동 생성

#### `/dk-plan`
기능 요청을 TDD 구조의 작업 체크리스트로 분해합니다.

- **코드 작업**: Red → Green → Refactor 사이클
- **비코드 작업**: Work → Verify 사이클 (아트, 레벨, 설정 등)
- 30분~2시간 단위의 작업 분할
- `features/{FeatureName}.md` 자동 생성

#### `/dk-feature`
개발 기능 목록을 `features.md`로 관리합니다.

- 기능 추가/제거/수정/완료 처리 지원
- 자동 번호 부여 (FEAT-001, FEAT-002, …)
- 우선순위 기반 목록 정렬

#### `/dk-bugfix`
문제의 원인을 파악하고 수정하는 작업을 자동화합니다.

- 에러 메시지·스크린샷·로그 등 다양한 입력으로 원인 분석
- 코드 문제인 경우 회귀 방지 테스트 자동 추가 (Red → Green 확인)
- 수정 후 사용자 확인 단계 포함 — 미해결 시 변경사항 전체 롤백 후 재분석

---

## 커맨드 (Commands)

스킬을 조합해 복잡한 워크플로우를 한 번에 실행합니다.

#### `/dk-blueprint {게임명 또는 식별 정보}`
게임 분석 보고서와 GRD를 자동으로 순차 생성합니다.

```
/dk-blueprint Candy Crush Saga
/dk-blueprint https://play.google.com/store/apps/details?id=com.king.candycrushsaga
/dk-blueprint 퍼즐 게임, 화살표를 탭해서 탈출시키는 모바일 게임
```

- `dk-analysis` → `dk-grd` 순서로 스킬 자동 실행
- 출력: `{게임명}_analysis.md` + `{게임명}_GRD.md`

#### `/dk-implement [{기능 설명}]`
`features.md`의 다음 항목을 가져와 세부 계획을 수립하고, TDD 사이클에 따라 개발·테스트까지 완료합니다.

```
/dk-implement                          # features.md의 다음 항목 구현
/dk-implement 하트 자동 회복 기능 추가  # features.md에 추가 후 구현
```

- `dk-plan` 스킬로 작업 명세서 생성 후 사용자 승인
- Red → Green → Refactor 사이클로 구현
- 완료 후 `features.md` 완료 표시 자동 업데이트

#### `/dk-commit`
변경 내용을 분석하여 일관된 형식의 커밋 메시지를 작성하고 커밋합니다.

- 타입 자동 분류: `feat` / `fix` / `balance` / `ui` / `perf` / `refactor` / `chore`
- 스토어 업데이트 노트 추출에 최적화된 형식
- 커밋 전 메시지 초안을 사용자에게 확인 후 실행

---

## 개발 규칙 (Rules)

Claude Code가 코드 생성 및 설계 시 준수하는 가이드라인입니다.

### 아키텍처 (`mvp-architecture.md`)
- **2 Assembly 구조**: Core (Model + Presenter + 인터페이스) / Presentation (View + Infrastructure + Installer)
- **MVP 패턴**: Model (데이터 + 비즈니스 로직) / View (MonoBehaviour) / Presenter (순수 C#)
- 단방향 의존성: Presentation → Core
- View ↔ Presenter 통신: R3 Observable (인터페이스 기반)
- Presenter 간 외부 알림: MessagePipe
- DI: VContainer

### 코드 스타일 (`code-style.md`)
- PascalCase 클래스, camelCase 변수, `_prefix` 프라이빗 필드
- Early return으로 중첩 최소화
- UniTask (코루틴 대체), Awake에서 컴포넌트 캐싱
- XML 문서 주석 사용 금지

### TDD 워크플로우 (`tdd-workflow.md`)
- 엄격한 **Red → Green → Refactor** 사이클 준수
- 테스트 없이 구현 금지 (테스트 우선)
- EditMode 테스트: Core 레이어 (빠른 실행)
- PlayMode 테스트: View/Infrastructure 레이어 (느린 실행)
- NSubstitute 목킹 라이브러리 사용

---

## 권장 사용 흐름

```
[프로젝트 초기 설정]
1. /project-context          → 프로젝트 분석 및 project-context.md 생성

[게임 기획]
2. /dk-blueprint {게임명}     → 참조 게임 분석 보고서 + GRD 자동 생성
                               (dk-analysis → dk-grd 순차 실행)

[기능 개발]
3. /dk-feature {기능 설명}   → 개발할 기능 목록에 추가
4. /dk-implement             → 다음 기능 계획 수립 후 TDD로 구현
5. /dk-commit                → 변경사항 커밋 메시지 표준화 후 커밋
```

---

## 플러그인 구조

```
game-development/
├── .claude-plugin/
│   └── plugin.json              # 플러그인 매니페스트
├── skills/
│   ├── project-context/         # 프로젝트 분석 → project-context.md
│   ├── dk-analysis/             # 게임 분석 보고서 생성
│   ├── dk-grd/                  # 게임 요구사항 명세서(GRD) 생성
│   ├── dk-plan/                 # 기능 작업 체크리스트 생성
│   ├── dk-feature/              # 기능 목록 관리
│   └── dk-bugfix/               # 버그 원인 파악 및 수정, 회귀 테스트 추가
├── rules/
│   ├── mvp-architecture.md      # MVP 아키텍처 가이드
│   ├── code-style.md            # C#/Unity 코드 스타일
│   └── tdd-workflow.md          # TDD 워크플로우
└── commands/
    ├── dk-blueprint.md          # 분석 보고서 + GRD 자동 생성
    ├── dk-implement.md          # 기능 구현 (계획~테스트 일괄)
    └── dk-commit.md             # 커밋 메시지 표준화
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
