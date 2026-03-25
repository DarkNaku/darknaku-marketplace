# darknaku-marketplace

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-purple.svg)](https://code.claude.com)
[![Version](https://img.shields.io/badge/Version-0.0.1-green.svg)]()
[![Author](https://img.shields.io/badge/Author-DarkNaku-orange.svg)](https://github.com/darknaku)

> **체계적인 게임 개발을 위한 Claude Code 플러그인 마켓플레이스**

darknaku-marketplace는 Claude Code용 플러그인 마켓플레이스입니다. 현재 Unity 기반 게임 개발의 전체 사이클을 AI와 함께 체계적으로 진행할 수 있도록 설계된 **Game Development** 플러그인을 제공합니다.

---

## 플러그인 목록

| 플러그인 | 버전 | 설명 |
|---------|:----:|------|
| **Game Development** | 0.0.1 | 체계적인 게임 개발을 위한 스킬 및 규칙 모음 |

---

## Game Development 플러그인

Unity + C# 환경에서 **분석 → 기획 → 설계 → 구현**의 전체 개발 사이클을 AI와 함께 체계적으로 진행할 수 있는 스킬과 개발 가이드라인을 제공합니다.

### 스킬 (Skills)

#### 1. Project Analyst (`/project-analyst`)
기존 Unity 프로젝트를 분석하여 Claude Code가 참조할 수 있는 컨텍스트 문서를 자동 생성합니다.

- 폴더 구조, 패키지 의존성, 아키텍처 레이어 추출
- DI 패턴, 코딩 컨벤션, 네이밍 규칙 식별
- `PROJECT_CONTEXT.md` 자동 생성

#### 2. Game Analysis Reporter (`/game-analysis`)
참조 게임을 웹 검색을 통해 분석하고 MDA 프레임워크 기반의 표준화된 게임 분석 문서를 생성합니다.

- MDA 프레임워크 (Mechanics / Dynamics / Aesthetics) 기반 분석
- 플레이어 경험 목표, 게임 루프, 수익화 모델 포함
- `{GameName}_analysis.md` 자동 생성

#### 3. Game PRD Writer (`/game-prd`)
게임 분석 리포트를 기반으로 구현 가능한 수준의 PRD(제품 요구사항 문서)를 작성합니다.

- 디자인 필러, 핵심 게임플레이 루프 정의
- 시스템 명세, 콘텐츠 정의, UX/UI 스펙 포함
- MVP 범위 및 스프린트 계획 포함
- `{GameName}_PRD.md` 자동 생성

#### 4. Feature Plan (`/feature-plan`)
기능 요청을 TDD 구조의 작업 체크리스트로 분해합니다.

- **코드 작업**: Red → Green → Refactor 사이클
- **비코드 작업**: Work → Verify 사이클 (아트, 레벨, 설정 등)
- 30분~2시간 단위의 작업 분할
- 의존성 순서 및 통합 테스트 작업 포함
- `features/{FeatureName}.md` 자동 생성

#### 5. Feature Manager (`/feature`)
개발 기능 목록을 `features.md`로 관리합니다.

- 기능 추가/제거/수정/완료 처리 지원
- 자동 번호 부여 (FEAT-001, FEAT-002, …)
- 우선순위 기반 목록 정렬

---

### 커맨드 (Commands)

스킬을 조합해 복잡한 워크플로우를 한 번에 실행하는 커맨드입니다.

#### `/analysis {게임명 또는 식별 정보}`
게임 분석 보고서와 PRD를 자동으로 순차 생성합니다.

```
/analysis Candy Crush Saga
/analysis https://play.google.com/store/apps/details?id=com.king.candycrushsaga
/analysis 퍼즐 게임, 화살표를 탭해서 탈출시키는 모바일 게임
```

- `game-analysis` → `game-prd` 순서로 스킬 자동 실행
- 출력: `{게임명}_analysis.md` + `{게임명}_PRD.md`

#### `/implement [{기능 설명}]`
`features.md`의 다음 항목을 가져와 세부 계획을 수립하고, TDD 사이클에 따라 개발·테스트까지 완료합니다.

```
/implement                          # features.md의 다음 항목 구현
/implement 하트 자동 회복 기능 추가  # features.md에 추가 후 구현
```

- `feature-plan` 스킬로 작업 명세서 생성 후 사용자 승인
- Red → Green → Refactor 사이클로 구현
- 완료 후 `features.md` 완료 표시 자동 업데이트

#### `/commit`
변경 내용을 분석하여 일관된 형식의 커밋 메시지를 작성하고 커밋합니다.

- 타입 자동 분류: `feat` / `fix` / `balance` / `ui` / `perf` / `refactor` / `chore`
- 스토어 업데이트 노트 추출에 최적화된 형식 (`feat`, `fix`, `balance`, `ui`, `perf` 만 필터링)
- 커밋 전 메시지 초안을 사용자에게 확인 후 실행

---

## 개발 규칙 (Rules)

플러그인에는 Claude Code가 준수해야 할 개발 가이드라인이 포함되어 있습니다.

### 아키텍처 (`architecture.md`)
- **2레이어 구조**: Presentation + Core
- **DDD-Lite 패턴**: Domain / Application(UseCase) / Infrastructure
- **MVP 패턴**: View (MonoBehaviour) + Presenter (pure C#) + View Interface
- 단방향 의존성: Presentation → Core
- Core 레이어는 Unity 런타임 타입 참조 금지
- VContainer (DI), R3 (반응형 상태 바인딩), MessagePipe (pub/sub)
- `Result<T>` 타입으로 에러 처리 (도메인에서 예외 금지)

### 코드 스타일 (`code-style.md`)
- PascalCase 클래스, camelCase 변수, `_prefix` 프라이빗 필드
- Early return으로 중첩 최소화
- 명시적인 경우 `var` 키워드 사용
- UniTask (코루틴 대체), Awake에서 컴포넌트 캐싱
- XML 문서 주석 사용 금지

### TDD 워크플로우 (`tdd-workflow.md`)
- 엄격한 **Red → Green → Refactor** 사이클 준수
- 테스트 없이 구현 금지 (테스트 우선)
- EditMode 테스트: Core/UseCase 레이어 (빠른 실행)
- PlayMode 테스트: View/Infrastructure 레이어 (느린 실행)
- 페이크 레포지토리 (인메모리 구현)로 테스트
- NSubstitute 목킹 라이브러리 사용
- 커밋 메시지에 **[구조]** 또는 **[동작]** 변경 구분 명시
- 테스트 메서드명은 한국어 + 언더스코어 (예: `양수_두개를_더하면_올바른_합을_반환한다()`)

---

## 설치

### 마켓플레이스 추가

```bash
/plugin marketplace add darknaku/darknaku-marketplace
```

### 플러그인 설치

```bash
/plugin install game-development
```

---

## 프로젝트 구조

```
darknaku-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # 마켓플레이스 메타데이터
├── plugins/
│   └── game-development/         # Game Development 플러그인
│       ├── .claude-plugin/
│       │   └── plugin.json       # 플러그인 매니페스트
│       ├── skills/               # 스킬 정의
│       │   ├── project-analyst/  # 프로젝트 분석
│       │   ├── game-analysis/    # 게임 분석 리포터
│       │   ├── game-prd/         # PRD 작성
│       │   ├── feature-plan/     # 기능 계획
│       │   └── feature/          # 기능 목록 관리
│       ├── rules/                # 개발 가이드라인
│       │   ├── architecture.md   # DDD-Lite 아키텍처
│       │   ├── code-style.md     # C#/Unity 코드 스타일
│       │   └── tdd-workflow.md   # TDD 워크플로우
│       └── commands/             # CLI 커맨드
│           ├── analysis.md       # 분석 보고서 + PRD 자동 생성
│           ├── implement.md      # 기능 구현 (계획~테스트 일괄)
│           └── commit.md         # 커밋 메시지 표준화
└── LICENSE
```

---

## 권장 사용 흐름

```
[프로젝트 초기 설정]
1. /project-analyst          → 기존 프로젝트 분석 및 PROJECT_CONTEXT.md 생성

[게임 기획]
2. /analysis {게임명}         → 참조 게임 분석 보고서 + PRD 자동 생성
                               (game-analysis → game-prd 순차 실행)

[기능 개발]
3. /feature {기능 설명}       → 개발할 기능 목록에 추가
4. /implement                → 다음 기능 계획 수립 후 TDD로 구현
5. /commit                   → 변경사항 커밋 메시지 표준화 후 커밋
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

---

## 라이선스

MIT License. 자세한 내용은 [LICENSE](LICENSE)를 참조하세요.

---

Made by [DarkNaku](https://github.com/darknaku)
