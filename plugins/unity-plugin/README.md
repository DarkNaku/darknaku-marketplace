# Unity Plugin

[![Version](https://img.shields.io/badge/Version-0.2.4-green.svg)]()

Unity + C# 환경에 특화된 스킬과 개발 가이드라인을 제공합니다. game-plugin의 범용 스킬과 함께 사용하여 Unity 프로젝트의 환경 분석, 코드 컨벤션 파악, 테스트 실행, UI 구현을 지원합니다.

---

## 사용 가이드

### A. 신규 프로젝트 — 처음부터 시작하기

참조 게임을 분석하고 PRD를 자동 생성한 뒤, 기능을 하나씩 구현해 나갑니다.

```
1. Unity 프로젝트 생성 후 플러그인 설치

2. 프로젝트 환경 분석
   → "프로젝트 분석해줘"              # game-env-analyzer → unity-env-analyzer 자동 위임

3. 게임 기획
   → /blueprint Candy Crush Saga     # 분석 보고서 + PRD 자동 생성 (game-plugin)

4. 기능 구현 (PRD의 기능 목록을 순서대로 처리)
   → /implement                      # 다음 기능 자동 선택 → 계획 → TDD 구현 → 커밋 → 머지
   → /implement                      # 반복...

5. 버그 발생 시
   → "이 버그 수정해줘: {증상}"       # game-bugfix 자동 발동 (game-plugin)
```

### B. 기존 프로젝트 — 이미 진행 중인 프로젝트에 도입하기

기존 코드베이스의 환경과 컨벤션을 먼저 분석한 뒤, 플러그인의 스킬을 활용합니다.

```
1. 플러그인 설치

2. 프로젝트 환경 분석
   → "프로젝트 분석해줘"              # game-env-analyzer → unity-env-analyzer 자동 위임

3. 코드 컨벤션 분석
   → "코드 컨벤션 분석해줘"           # unity-conventions-analyzer → UNITY_PROJECT_CONVENTIONS.md 생성

4. 기능 목록 준비 (아래 중 택 1)
   a) PRD가 있는 경우: 기능 목록 섹션이 있는 *prd*.md 파일을 프로젝트에 배치
   b) PRD가 없는 경우: "기능 추가해줘: {기능 설명}" → FEATURES.md 자동 생성

5. 기능 구현
   → /implement                      # 기능 목록에서 다음 항목 자동 선택 → 구현 (game-plugin)
   → /implement {새 기능 설명}        # 기능 목록에 추가 후 바로 구현 (game-plugin)

6. 개별 스킬 활용 (필요 시)
   → "이 기능 작업 분해해줘"           # game-plan → 작업 명세서 생성 (game-plugin)
   → "커밋해줘"                       # game-commit → 표준 커밋 (game-plugin)
   → "이 버그 수정해줘"               # game-bugfix → 가설 기반 수정 (game-plugin)
```

### 주요 차이점

| | 신규 프로젝트 | 기존 프로젝트 |
|---|---|---|
| **시작점** | `/blueprint`로 기획부터 | 환경·컨벤션 분석부터 |
| **기능 목록** | PRD 자동 생성 | 기존 PRD 활용 또는 FEATURES.md 생성 |
| **컨벤션 분석** | 코드가 없으므로 생략 | `unity-conventions-analyzer`로 기존 스타일 파악 |

---

## 스킬 (Skills)

| 스킬 | 설명 |
|------|------|
| [`unity-env-analyzer`](#unity-env-analyzer) | Unity 프로젝트 분석 → `DEV_ENV.md` 자동 생성 |
| [`unity-conventions-analyzer`](#unity-conventions-analyzer) | 코드 아키텍처·컨벤션 분석 → `UNITY_PROJECT_CONVENTIONS.md` 자동 생성 |
| [`unity-test`](#unity-test) | Unity Test Framework 기반 테스트 작성 및 실행 |
| [`unity-uitoolkit`](#unity-uitoolkit) | Unity UI Toolkit(UXML + USS + C#) 기반 UI 구현 가이드 |
| [`unity-vcontainer`](#unity-vcontainer) | VContainer DI 컨테이너 등록 패턴, LifetimeScope 설계, EntryPoint 활용 가이드 |
| [`unity-r3`](#unity-r3) | R3 Reactive Extensions 구독 패턴, ReactiveProperty, 오퍼레이터 활용 가이드 |
| [`unity-messagepipe`](#unity-messagepipe) | MessagePipe Pub/Sub, Request/Response 패턴, 시스템 간 통신 가이드 |
| [`unity-unitask`](#unity-unitask) | UniTask 비동기 패턴, 취소 처리, PlayerLoop 타이밍 가이드 |
| [`unity-zlinq`](#unity-zlinq) | ZLinq 제로 할당 LINQ, GameObject 트리 탐색, GC 최적화 가이드 |
| [`unity-dotween`](#unity-dotween) | DOTween 트위닝, Sequence 조합, 피드백 효과 가이드 |
| [`unity-inputsystem`](#unity-inputsystem) | Input System Action 설계, 바인딩, 콜백 처리, 리바인딩 가이드 |
| [`unity-addressables`](#unity-addressables) | Addressables 에셋 로딩/해제, AssetReference, 메모리 관리 가이드 |

---

#### `unity-env-analyzer`
Unity 프로젝트를 분석하여 개발 환경 문서를 자동 생성합니다.

- 패키지·라이브러리, 폴더 구조, Unity 버전 등 추출
- Unity 버전에 따른 C# 버전 자동 매핑
- `Docs/DEV_ENV.md` 자동 생성

#### `unity-conventions-analyzer`
Unity 프로젝트의 코드베이스를 분석하여 아키텍처와 코딩 컨벤션을 문서화합니다.

- 코드 아키텍처, 디자인 패턴, 네이밍 규칙 역추론
- `UNITY_PROJECT_CONVENTIONS.md` 자동 생성

#### `unity-test`
Unity 프로젝트에서 game-tdd 스킬의 TDD 사이클을 Unity Test Framework에 맞게 실행합니다.

- EditMode 테스트 우선, PlayMode는 Unity 런타임 필요 시에만
- Unity Test Runner 및 MCP 연동 지원
- 테스트 파일 구성 및 네이밍 규칙 제공

#### `unity-uitoolkit`
Unity 프로젝트에서 UI를 만들거나 수정할 때 UI Toolkit(UXML + USS + C#)을 사용하도록 안내합니다.

- Canvas/GameObject 기반 uGUI 사용 금지, UIDocument + UXML 구조 강제
- UXML 작성 규칙, USS 스타일링, C# View 패턴 안내
- MVP + VContainer + R3 연동 패턴 제공
- 팝업·동적 UI, 에디터 UI 패턴 포함

#### `unity-vcontainer`
VContainer를 사용하는 Unity 프로젝트에서 DI 컨테이너 활용 방법을 안내합니다.

- 생성자 주입 기본, LifetimeScope 계층 설계
- Register/RegisterComponent/RegisterEntryPoint 등록 패턴
- EntryPoint 라이프사이클 인터페이스 (IStartable, ITickable 등)
- 씬 간 스코프 관리, Additive Scene 로딩 패턴
- MVP + VContainer 연동, R3 연동 패턴

#### `unity-r3`
R3(Reactive Extensions)를 사용하는 Unity 프로젝트에서 리액티브 프로그래밍 패턴을 안내합니다.

- Observable, Subject, ReactiveProperty 사용 패턴
- 구독 관리 (AddTo, DisposableBag) 및 누수 방지
- 프레임 기반 오퍼레이터 (EveryUpdate, EveryValueChanged)
- 비동기 연동 (SelectAwait, AwaitOperation)
- VContainer + R3 + MVP 통합 패턴

#### `unity-messagepipe`
MessagePipe를 사용하는 Unity 프로젝트에서 디커플링된 메시지 통신 패턴을 안내합니다.

- IPublisher/ISubscriber 기반 Pub/Sub 패턴
- IRequestHandler 기반 Request/Response (Mediator) 패턴
- 키 기반 메시지 브로커, 비동기 Pub/Sub
- MessageHandlerFilter 미들웨어 체인
- VContainer + MessagePipe + R3 통합 아키텍처

#### `unity-unitask`
UniTask를 사용하는 Unity 프로젝트에서 async/await 비동기 패턴을 안내합니다.

- 코루틴 대체: async UniTask 기본 패턴
- CancellationToken 전파, 타임아웃, 취소 처리
- PlayerLoopTiming, Yield vs NextFrame 구분
- UniTask.WhenAll/WhenAny 작업 결합
- AsyncOperation, Addressables, DOTween 비동기 대기
- IUniTaskAsyncEnumerable, Channel 비동기 스트림

#### `unity-zlinq`
ZLinq를 사용하는 Unity 프로젝트에서 제로 할당 LINQ 패턴을 안내합니다.

- AsValueEnumerable() 제로 할당 LINQ 체인
- LINQ to Tree: Ancestors, Children, Descendants, OfComponent
- GetComponentsInChildren 대체 패턴
- UI Toolkit VisualElement 트리 탐색
- NativeArray/NativeList LINQ 지원
- Drop-In 소스 생성기로 기존 LINQ 자동 교체

#### `unity-dotween`
DOTween을 사용하는 Unity 프로젝트에서 트위닝 애니메이션 패턴을 안내합니다.

- Tweener 생성 (DOMove, DORotate, DOScale, DOFade 등)
- Sequence 조합 (Append, Join, Insert 타임라인)
- 설정 체인 (SetEase, SetLoops, SetLink 수명 관리)
- 피드백 효과 (Punch, Shake)
- UI 애니메이션, 게임 피드백 실전 패턴
- UniTask 연동 (WithCancellation)

#### `unity-inputsystem`
Input System을 사용하는 Unity 프로젝트에서 입력 처리 패턴을 안내합니다.

- Action 기반 입력 추상화 (Value, Button, Pass-Through)
- Action Map 설계 및 상태별 전환 패턴
- 프로젝트 전역 Action, InputActionAsset + C# 클래스 생성 워크플로우
- 폴링 / 콜백 입력 응답, Action Phase 활용
- Composite Binding (2DVector, Axis, Modifier)
- Interaction (Hold, Tap, MultiTap 등)
- PlayerInput 컴포넌트 Behavior 옵션 비교
- EnhancedTouch API, TouchSimulation
- 런타임 리바인딩, 바인딩 저장/복원, 표시 문자열
- VContainer 연동 입력 서비스 추상화 패턴

#### `unity-addressables`
Addressables를 사용하는 Unity 프로젝트에서 에셋 관리 패턴을 안내합니다.

- AssetReference Inspector 할당, 타입 제한 레퍼런스
- LoadAssetAsync, InstantiateAsync, LoadSceneAsync 로딩 패턴
- Release/ReleaseInstance 메모리 해제 규칙
- 레퍼런스 카운팅, AssetBundle 메모리 관리
- 프리로딩, 오브젝트 풀, 씬 전환 실전 패턴

---

## 개발 규칙 (Rules)

Claude Code가 코드 생성 및 설계 시 준수하는 가이드라인입니다.

### 코드 스타일 (`code-style.md`)
- PascalCase 클래스, camelCase 변수, `_prefix` 프라이빗 필드
- Early return으로 중첩 최소화
- UniTask (코루틴 대체), Awake에서 컴포넌트 캐싱
- XML 문서 주석 사용 금지

---

## 플러그인 구조

```
unity-plugin/
├── .claude-plugin/
│   └── plugin.json                  # 플러그인 매니페스트
├── skills/
│   ├── unity-env-analyzer/          # Unity 개발 환경 분석
│   ├── unity-conventions-analyzer/  # 코드 컨벤션 분석
│   ├── unity-test/                  # Unity Test Framework 테스트
│   ├── unity-uitoolkit/             # Unity UI Toolkit 기반 UI 구현 가이드
│   ├── unity-vcontainer/            # VContainer DI 컨테이너 활용 가이드
│   ├── unity-r3/                    # R3 Reactive Extensions 활용 가이드
│   ├── unity-messagepipe/           # MessagePipe 메시지 브로커 활용 가이드
│   ├── unity-unitask/               # UniTask 비동기 처리 가이드
│   ├── unity-zlinq/                 # ZLinq 제로 할당 LINQ 가이드
│   ├── unity-dotween/               # DOTween 트위닝 애니메이션 가이드
│   ├── unity-inputsystem/           # Input System 입력 처리 가이드
│   └── unity-addressables/          # Addressables 에셋 관리 가이드
└── rules/
    └── code-style.md                # C#/Unity 코드 스타일
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
| **입력** | Input System |
| **UI** | UI Toolkit, MonoBehaviour 기반 MVP |
