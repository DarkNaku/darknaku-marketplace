# PROJECT_CONTEXT.md

> **생성 일자**: {YYYY-MM-DD}  
> **프로젝트**: {프로젝트명}  
> **분석 대상 경로**: {루트 경로}  
> **목적**: Claude Code가 기능 구현 시 이 프로젝트의 컨벤션과 패턴을 즉시 참조하기 위한 문서

---

## 목차

1. [개발 환경](#1-개발-환경)
2. [패키지 & 라이브러리](#2-패키지--라이브러리)
3. [폴더 구조](#3-폴더-구조)
4. [아키텍처 레이어](#4-아키텍처-레이어)
5. [코딩 스타일 & 네이밍 규칙](#5-코딩-스타일--네이밍-규칙)
6. [핵심 패턴](#6-핵심-패턴)
7. [테스트 구조 & 규칙](#7-테스트-구조--규칙)
8. [새 기능 추가 체크리스트](#8-새-기능-추가-체크리스트)

---

## 1. 개발 환경

| 항목 | 값 |
|------|-----|
| 엔진/런타임 | {예: Unity 6000.0.23f1} |
| 스크립팅 백엔드 | {예: IL2CPP / Mono} |
| API 호환성 레벨 | {예: .NET Standard 2.1} |
| 타겟 플랫폼 | {예: Android, iOS} |
| 프로젝트명 | {ProductName} |
| 회사명 | {CompanyName} |

---

## 2. 패키지 & 라이브러리

### 2.1 핵심 패키지 (기능별 분류)

| 역할 | 패키지명 | 버전 | 비고 |
|------|----------|------|------|
| DI 컨테이너 | {예: VContainer} | {버전} | |
| 이벤트 버스 | {예: MessagePipe} | {버전} | |
| 비동기 | {예: UniTask} | {버전} | |
| 반응형 | {예: R3} | {버전} | |
| 트위닝 | {예: DOTween} | {버전} | |
| 테스트 Mock | {예: Moq} | {버전} | |
| UI 프레임워크 | {예: UI Toolkit} | {버전} | |

### 2.2 전체 패키지 목록

```json
{
  "dependencies": {
    // manifest.json 내용 그대로 붙여넣기
  }
}
```

### 2.3 패키지 사용 원칙
- {예: 비동기 작업은 반드시 UniTask 사용. Task/Thread 직접 사용 금지.}
- {예: 이벤트 발행/구독은 MessagePipe를 통해서만 한다.}
- {예: 애니메이션은 DOTween 사용. 코루틴 기반 애니메이션 금지.}

---

## 3. 폴더 구조

### 3.1 전체 구조

```
{프로젝트 루트}/
├── Assets/
│   ├── Scripts/
│   │   ├── {레이어1}/          # {설명}
│   │   ├── {레이어2}/          # {설명}
│   │   ├── {레이어3}/          # {설명}
│   │   └── {레이어4}/          # {설명}
│   ├── Tests/
│   │   ├── EditMode/           # 순수 C# 단위 테스트
│   │   └── PlayMode/           # Unity 런타임 테스트
│   ├── Resources/
│   ├── Prefabs/
│   └── ...
├── Packages/
│   └── manifest.json
└── ProjectSettings/
```

### 3.2 Scripts 내부 구조 상세

```
Scripts/
├── {Feature 또는 레이어명}/
│   ├── Domain/                 # 순수 C# — Unity 의존 없음
│   │   ├── Models/             # Entity, ValueObject
│   │   ├── Repositories/       # 인터페이스만
│   │   └── Services/           # 도메인 서비스
│   ├── Application/            # UseCase, 조율 로직
│   ├── Infrastructure/         # 외부 연동, Repository 구현체
│   └── Presentation/           # MonoBehaviour, View, Presenter
```

### 3.3 새 기능 파일 위치 규칙
- 도메인 로직 → `Scripts/{Feature}/Domain/`
- 유스케이스 → `Scripts/{Feature}/Application/`
- UI/MonoBehaviour → `Scripts/{Feature}/Presentation/`
- 테스트 → `Tests/EditMode/{Feature}/`

---

## 4. 아키텍처 레이어

### 4.1 레이어 구조 다이어그램

```
┌─────────────────────────────┐
│      Presentation Layer      │  MonoBehaviour, View, Presenter
│   (Unity 의존 허용)          │  ← UniTask, R3, VContainer
└──────────────┬──────────────┘
               │ 참조 (단방향)
┌──────────────▼──────────────┐
│      Application Layer       │  UseCase, ApplicationService
│   (순수 C# + UniTask 허용)   │
└──────────────┬──────────────┘
               │ 참조 (단방향)
┌──────────────▼──────────────┐
│        Domain Layer          │  Entity, ValueObject, IRepository
│   (순수 C# — 의존 없음)      │
└─────────────────────────────┘
               ▲
┌──────────────┴──────────────┐
│    Infrastructure Layer      │  Repository 구현체, 외부 API
│   (Unity/외부 라이브러리)    │
└─────────────────────────────┘
```

### 4.2 의존성 규칙
- Domain은 어떤 레이어도 참조하지 않는다.
- Application은 Domain만 참조한다.
- Infrastructure는 Domain 인터페이스를 구현한다.
- Presentation은 Application을 통해서만 Domain에 접근한다.
- **레이어 간 역방향 참조 금지** — 위반 시 인터페이스로 역전.

### 4.3 어셈블리 정의(.asmdef) 구조

| asmdef 파일 | 참조 허용 |
|-------------|-----------|
| `{Project}.Domain` | 없음 |
| `{Project}.Application` | Domain |
| `{Project}.Infrastructure` | Domain, Application |
| `{Project}.Presentation` | Application, (Infrastructure는 DI로만) |
| `{Project}.Tests.EditMode` | Domain, Application |

---

## 5. 코딩 스타일 & 네이밍 규칙

### 5.1 네이밍 규칙

| 대상 | 규칙 | 예시 ✅ | 예시 ❌ |
|------|------|---------|---------|
| 클래스 | PascalCase | `PlayerPresenter` | `playerPresenter` |
| 인터페이스 | `I` + PascalCase | `IPlayerRepository` | `PlayerRepository` (인터페이스) |
| UseCase | `{동사}{명사}UseCase` | `AttackEnemyUseCase` | `AttackUseCase` |
| Repository 인터페이스 | `I{명사}Repository` | `IPlayerRepository` | |
| Presenter | `{명사}Presenter` | `PlayerPresenter` | |
| View | `{명사}View` | `PlayerView` | |
| 메서드 | PascalCase + 동사 시작 | `GetPlayer()`, `ExecuteAsync()` | `player()` |
| 비동기 메서드 | `{동사}Async` 접미사 | `LoadDataAsync()` | `LoadData()` (비동기인데) |
| private 필드 | `_camelCase` | `_playerData` | `playerData`, `m_playerData` |
| public 프로퍼티 | PascalCase | `PlayerName` | `playerName` |
| 이벤트/메시지 | `{명사}{과거동사}` 또는 `On{동사}` | `PlayerDied`, `OnAttack` | |
| 상수 | PascalCase 또는 ALL_CAPS | `MaxHealth` / `MAX_HEALTH` | |
| 테스트 메서드 | `{테스트대상}_{조건}_{기대결과}` | `Attack_WhenEnemyDead_ThrowsException` | `TestAttack` |

### 5.2 코드 스타일

```csharp
// ✅ 이 프로젝트의 표준 스타일
public class PlayerPresenter : IDisposable
{
    // 필드: private _ 접두사
    private readonly IPlayerRepository _playerRepository;
    private readonly CompositeDisposable _disposables = new();

    // 생성자 DI
    public PlayerPresenter(IPlayerRepository playerRepository)
    {
        _playerRepository = playerRepository;
    }

    // 비동기 메서드: UniTask + Async 접미사
    public async UniTask InitializeAsync(CancellationToken ct)
    {
        var player = await _playerRepository.GetAsync(ct);
        // ...
    }

    public void Dispose() => _disposables.Dispose();
}
```

### 5.3 금지 패턴

```csharp
// ❌ 금지: Task 직접 사용
public async Task LoadAsync() { }          // UniTask 사용할 것

// ❌ 금지: FindObjectOfType, GameObject.Find
var player = FindObjectOfType<Player>();   // DI로 주입받을 것

// ❌ 금지: 싱글톤 직접 접근
GameManager.Instance.DoSomething();        // DI 컨테이너 통해 주입

// ❌ 금지: Presentation에서 Domain 직접 접근
// PlayerView에서 PlayerRepository 직접 참조 금지
```

---

## 6. 핵심 패턴

### 6.1 DI 등록 패턴 (VContainer)

```csharp
// LifetimeScope 예시 — 실제 프로젝트 코드 기반
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // Domain
        builder.Register<IPlayerRepository, PlayerRepository>(Lifetime.Singleton);

        // Application
        builder.Register<AttackEnemyUseCase>(Lifetime.Transient);

        // Presentation (EntryPoint)
        builder.RegisterEntryPoint<PlayerPresenter>();

        // MessagePipe
        builder.RegisterMessagePipe();
        builder.RegisterMessageBroker<PlayerDiedMessage>();
    }
}
```

### 6.2 이벤트 패턴 (MessagePipe)

```csharp
// 메시지 정의
public record PlayerDiedMessage(int PlayerId, Vector3 Position);

// 발행 (Publisher)
public class PlayerUseCase
{
    private readonly IPublisher<PlayerDiedMessage> _publisher;

    public PlayerUseCase(IPublisher<PlayerDiedMessage> publisher)
    {
        _publisher = publisher;
    }

    public void Die(int playerId)
    {
        _publisher.Publish(new PlayerDiedMessage(playerId, Vector3.zero));
    }
}

// 구독 (Subscriber)
public class PlayerPresenter : IStartable, IDisposable
{
    private readonly ISubscriber<PlayerDiedMessage> _subscriber;
    private readonly CompositeDisposable _disposables = new();

    public void Start()
    {
        _subscriber.Subscribe(msg => OnPlayerDied(msg))
                   .AddTo(_disposables);
    }

    public void Dispose() => _disposables.Dispose();
}
```

### 6.3 비동기 패턴 (UniTask)

```csharp
// CancellationToken 항상 전달
public async UniTask<PlayerData> LoadPlayerAsync(CancellationToken ct)
{
    await UniTask.Delay(100, cancellationToken: ct);
    return await _repository.GetAsync(ct);
}

// 취소 처리
private CancellationTokenSource _cts;

private void OnDestroy()
{
    _cts?.Cancel();
    _cts?.Dispose();
}
```

### 6.4 반응형 패턴 (R3)

```csharp
// ReactiveProperty 사용
public class PlayerModel
{
    public ReactiveProperty<int> Health { get; } = new(100);
    public ReadOnlyReactiveProperty<bool> IsDead { get; }

    public PlayerModel()
    {
        IsDead = Health.Select(h => h <= 0).ToReadOnlyReactiveProperty();
    }
}

// View에서 바인딩
public class PlayerView : MonoBehaviour
{
    private void Start()
    {
        _model.Health
              .Subscribe(h => _healthText.text = h.ToString())
              .AddTo(destroyCancellationToken);
    }
}
```

### 6.5 Result 패턴 (있는 경우)

```csharp
// 성공/실패를 명시적으로 반환
public Result<PlayerData> GetPlayer(int id)
{
    if (id <= 0) return Result.Failure<PlayerData>("Invalid ID");
    return Result.Success(new PlayerData(id));
}

// 사용
var result = GetPlayer(playerId);
if (result.IsSuccess)
    // 성공 처리
else
    Debug.LogError(result.Error);
```

### 6.6 Repository 패턴

```csharp
// 인터페이스 (Domain 레이어)
public interface IPlayerRepository
{
    UniTask<PlayerData> GetAsync(int id, CancellationToken ct = default);
    UniTask SaveAsync(PlayerData player, CancellationToken ct = default);
}

// 구현체 (Infrastructure 레이어)
public class PlayerRepository : IPlayerRepository
{
    public async UniTask<PlayerData> GetAsync(int id, CancellationToken ct)
    {
        // 실제 데이터 접근 로직
        await UniTask.Yield(ct);
        return new PlayerData(id);
    }
}
```

---

## 7. 테스트 구조 & 규칙

### 7.1 테스트 구조

```
Tests/
├── EditMode/                   # 순수 C# — Unity 런타임 불필요
│   ├── Domain/                 # 도메인 로직 테스트
│   │   └── {Feature}/
│   │       └── {ClassName}Tests.cs
│   └── Application/            # UseCase 테스트
│       └── {Feature}/
│           └── {UseCaseName}Tests.cs
└── PlayMode/                   # Unity 런타임 필요
    └── {Feature}/
        └── {ClassName}Tests.cs
```

### 7.2 테스트 작성 규칙

```csharp
// 테스트 클래스 네이밍: {대상클래스}Tests
public class AttackEnemyUseCaseTests
{
    // Arrange 공통 요소는 필드로
    private Mock<IEnemyRepository> _mockRepository;
    private AttackEnemyUseCase _sut;  // System Under Test

    [SetUp]
    public void SetUp()
    {
        _mockRepository = new Mock<IEnemyRepository>();
        _sut = new AttackEnemyUseCase(_mockRepository.Object);
    }

    // 메서드명: {테스트대상}_{조건}_{기대결과}
    [Test]
    public async Task Execute_WhenEnemyExists_ReturnsSuccess()
    {
        // Arrange
        _mockRepository.Setup(r => r.GetAsync(1, default))
                       .ReturnsAsync(new EnemyData(1, 100));

        // Act
        var result = await _sut.ExecuteAsync(1);

        // Assert
        Assert.IsTrue(result.IsSuccess);
    }

    [Test]
    public void Execute_WhenEnemyNotFound_ThrowsNotFoundException()
    {
        // Arrange
        _mockRepository.Setup(r => r.GetAsync(99, default))
                       .ReturnsAsync((EnemyData)null);

        // Act & Assert
        Assert.ThrowsAsync<NotFoundException>(() =>
            _sut.ExecuteAsync(99).AsTask());
    }
}
```

### 7.3 Mock 프레임워크
- **사용**: {예: Moq 4.x}
- **패턴**: `Mock<T>` 생성 → `Setup()` → `Object` 주입
- **검증**: `Verify()` 로 호출 확인

### 7.4 테스트 원칙
- Domain 및 Application 레이어는 반드시 EditMode 테스트 작성
- Presentation(MonoBehaviour)은 PlayMode 또는 테스트 생략 가능
- 외부 의존성(Repository, API)은 반드시 Mock 처리
- 테스트당 단일 Assert 원칙 (가능한 경우)

---

## 8. 새 기능 추가 체크리스트

새 기능 `{FeatureName}`을 추가할 때 이 순서로 진행한다:

### 8.1 파일 생성 위치

```
✅ Domain 모델
   Scripts/{FeatureName}/Domain/Models/{FeatureName}Data.cs

✅ Repository 인터페이스
   Scripts/{FeatureName}/Domain/Repositories/I{FeatureName}Repository.cs

✅ UseCase
   Scripts/{FeatureName}/Application/{Action}{FeatureName}UseCase.cs

✅ Repository 구현체
   Scripts/{FeatureName}/Infrastructure/{FeatureName}Repository.cs

✅ Presenter
   Scripts/{FeatureName}/Presentation/{FeatureName}Presenter.cs

✅ View (필요 시)
   Scripts/{FeatureName}/Presentation/{FeatureName}View.cs

✅ DI 등록
   Scripts/Installer/{FeatureName}LifetimeScope.cs (또는 기존 Scope에 추가)

✅ 테스트
   Tests/EditMode/{FeatureName}/{FeatureName}UseCaseTests.cs
```

### 8.2 DI 등록 순서

```csharp
// 1. Domain Repository 인터페이스 → Infrastructure 구현체
builder.Register<I{Feature}Repository, {Feature}Repository>(Lifetime.Singleton);

// 2. UseCase
builder.Register<{Action}{Feature}UseCase>(Lifetime.Transient);

// 3. Presenter (EntryPoint로 등록)
builder.RegisterEntryPoint<{Feature}Presenter>();

// 4. 필요한 MessagePipe 메시지 등록
builder.RegisterMessageBroker<{Feature}ChangedMessage>();
```

### 8.3 구현 전 확인 사항
- [ ] Domain 레이어에 Unity 의존성이 없는가?
- [ ] UseCase가 단일 책임을 지키는가?
- [ ] 비동기 메서드에 `CancellationToken`이 전달되는가?
- [ ] 이벤트는 MessagePipe를 통하는가?
- [ ] Repository 인터페이스는 Domain에 있는가?
- [ ] 테스트 파일이 함께 생성되었는가?

---

*본 문서는 project-analyst 스킬로 자동 생성되었습니다. 프로젝트 구조 변경 시 재실행하여 최신 상태를 유지하세요.*
*재실행 명령: Claude Code에서 "프로젝트 다시 분석해줘" 입력*
