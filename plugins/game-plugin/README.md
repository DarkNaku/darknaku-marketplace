# Game Plugin

[![Version](https://img.shields.io/badge/Version-0.2.2-green.svg)]()

개발 환경(엔진·언어)에 독립적인 게임 개발 플러그인입니다. **기획·분석·설계·구현·커밋·버그 수정**까지 전체 개발 사이클을 지원합니다.

---

## 시작 가이드

플러그인 설치 후 프로젝트에서 처음 사용할 때 `game-init`을 먼저 실행합니다.

```
→ "초기화해줘"    # game-init 실행
```

`game-init`은 다음을 자동으로 수행합니다:

1. 프로젝트 환경 분석 (`Docs/DEV_ENV.md` 생성 — 없는 경우)
2. 감지된 환경에 맞는 코딩 스타일 규칙을 `.claude/rules/`에 링크

이후 기획부터 시작하려면 `/blueprint`, 바로 구현하려면 `/implement`를 사용합니다.

---

## 스킬 (Skills)

| 스킬 | 설명 |
|------|------|
| [`game-init`](#game-init) | 개발 환경에 맞는 규칙 파일을 `.claude/rules/`에 링크 |
| [`game-analysis`](#game-analysis) | 참조 게임 MDA 분석 보고서 생성 |
| [`game-prd`](#game-prd) | 분석 보고서 기반 PRD(제품 요구사항 명세서) 작성 |
| [`game-feature`](#game-feature) | PRD / FEATURES.md 기반 기능 목록 관리 |
| [`game-env-analyzer`](#game-env-analyzer) | 프로젝트 환경 자동 감지 및 `DEV_ENV.md` 생성 |
| [`game-plan`](#game-plan) | 기능 요청 → 작업 체크리스트 분해 |
| [`game-tdd`](#game-tdd) | TDD(Red → Green → Refactor) 사이클 기반 구현 |
| [`game-bugfix`](#game-bugfix) | 버그 원인 파악 · 수정 · 안전 장치 추가 |
| [`game-commit`](#game-commit) | 변경 내용 분석 → 일관된 형식의 커밋 메시지 작성 및 커밋 |

---

#### `game-init`
프로젝트의 개발 환경을 감지하고, 설치된 플러그인의 규칙 파일을 프로젝트에 연결합니다.

- `Docs/DEV_ENV.md`로 환경 감지 (없으면 `game-env-analyzer` 자동 실행)
- 감지된 환경에 맞는 플러그인의 `rules/` 파일을 탐색
- `.claude/rules/`에 심볼릭 링크로 연결 (`{플러그인명}--{파일명}` 형식)
- game-plugin의 규칙은 환경과 무관하게 항상 포함

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

#### `game-env-analyzer`
프로젝트 폴더를 분석하여 개발 환경 문서를 자동 생성합니다.

- Unity, Unreal, Godot, Phaser, Love2D, Cocos2d 등 게임 엔진 자동 감지
- Node.js, Python, Rust, Go 등 일반 개발 환경도 지원
- 전용 분석 스킬이 있는 엔진이 감지되면 해당 스킬을 자동 실행
- `Docs/DEV_ENV.md` 자동 생성

#### `game-plan`
기능 요청을 작업 체크리스트로 분해합니다.

- DEV_ENV.md 기반 프로젝트 환경 반영
- 의미 있는 작은 단위로 태스크 분해
- 30분~2시간 단위의 작업 분할
- `Docs/Features/{FeatureName}.md` 자동 생성

#### `game-tdd`
켄트 벡의 TDD와 Tidy First 원칙을 따라 기능을 구현합니다.

- 엄격한 **Red → Green → Refactor** 사이클 준수
- Vertical Slice 방식 (테스트 하나 → 구현 하나)
- 구조적 변경과 동작적 변경 분리 (Tidy First)

#### `game-bugfix`
버그의 근본 원인을 가설 기반으로 탐색하고 수정합니다.

- 가설 → 검증 → 수정 → 안전 장치 흐름
- 코드 문제: 회귀 테스트 자동 추가
- 비코드 문제: 재발 가능성 판단 후 작업 절차 안내
- 수정 실패 시 변경사항 롤백 후 재분석

#### `game-commit`
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
- feature 브랜치 생성 → `game-plan`으로 작업 명세서 생성 → 사용자 승인
- `game-tdd` 사이클로 구현
- 전체 테스트 통과 후 `game-commit`으로 최종 커밋 메시지 작성
- 원래 브랜치에 머지

---

## 플러그인 구조

```
game-plugin/
├── .claude-plugin/
│   └── plugin.json                  # 플러그인 매니페스트
├── skills/
│   ├── game-init/                   # 환경별 규칙 파일 링크
│   ├── game-analysis/               # 게임 분석 보고서 생성
│   ├── game-prd/                    # PRD(제품 요구사항 명세서) 생성
│   ├── game-feature/                # 기능 목록 관리
│   ├── game-env-analyzer/           # 개발 환경 자동 감지 및 문서화
│   ├── game-plan/                   # 기능 작업 체크리스트 생성
│   ├── game-tdd/                    # TDD 사이클 기반 구현
│   ├── game-bugfix/                 # 버그 원인 파악 및 수정
│   └── game-commit/                 # 커밋 메시지 표준화 및 커밋
└── commands/
    ├── blueprint.md                 # 분석 보고서 + PRD 자동 생성
    └── implement.md                 # 기능 구현 (계획~머지 일괄)
```
