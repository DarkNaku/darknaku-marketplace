---
name: unity-vcontainer
description: >
  VContainer(DI 컨테이너)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 등록 패턴, LifetimeScope 설계, EntryPoint 활용 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - VContainer, LifetimeScope, IContainerBuilder 언급 시
  - DI 컨테이너, 의존성 주입, IoC 관련 작업
  - Register, Resolve, EntryPoint, IStartable, ITickable 관련 작업
  - 씬 간 스코프 관리, 부모-자식 LifetimeScope 설계

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - VContainer를 사용하지 않는 프로젝트 (Zenject, Pure DI 등)
  - DI와 무관한 순수 게임플레이 로직
user-invocable: false
---

# Unity VContainer Skill

## 핵심 원칙

> **생성자 주입을 기본으로 사용한다.**
> Method Injection, Property Injection은 생성자 주입이 불가능한 경우(MonoBehaviour 등)에만 허용한다.
> 서비스 로케이터 패턴(`container.Resolve<T>()`를 직접 호출)은 팩토리 등록 내부를 제외하고 사용하지 않는다.

공식 문서: https://vcontainer.hadashikick.jp

---

## LifetimeScope 설계

### 기본 구조

모든 DI 등록은 `LifetimeScope`를 상속한 클래스의 `Configure` 메서드에서 수행한다.

```csharp
using VContainer;
using VContainer.Unity;

public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 서비스 등록
        builder.Register<ScoreService>(Lifetime.Singleton);

        // EntryPoint 등록
        builder.RegisterEntryPoint<GamePresenter>();
    }
}
```

### 스코프 계층 설계

씬 단위로 LifetimeScope를 분리하고, 부모-자식 관계로 의존성을 공유한다.

```
RootLifetimeScope          ← 앱 전역 (Singleton 서비스, 설정)
├── TitleLifetimeScope     ← 타이틀 씬
├── GameLifetimeScope      ← 게임플레이 씬
│   └── UILifetimeScope    ← UI 전용 (선택)
└── ResultLifetimeScope    ← 결과 씬
```

**원칙:**
- `RootLifetimeScope`는 `DontDestroyOnLoad` 씬에 배치하고 앱 전역 서비스를 등록한다
- 씬별 LifetimeScope는 해당 씬에서만 필요한 의존성을 등록한다
- 부모 스코프에 등록된 Singleton은 자식 스코프에서 자동으로 Resolve된다

---

## 등록 패턴

### Lifetime 선택 기준

| Lifetime | 용도 | 예시 |
|---|---|---|
| `Singleton` | 앱 전역 또는 스코프 내 단일 인스턴스 | `SaveManager`, `AudioService`, `NetworkClient` |
| `Scoped` | LifetimeScope당 하나의 인스턴스 | `GameState`, `LevelController` |
| `Transient` | Resolve할 때마다 새 인스턴스 | DTO, 일회성 핸들러 |

**기본은 `Scoped`다.** 명확한 이유가 없으면 `Scoped`를 사용한다.
`Transient`는 상태가 없고 매번 새 인스턴스가 필요할 때만 사용한다.

### 순수 C# 타입 등록

```csharp
// 구체 타입으로 등록
builder.Register<ScoreService>(Lifetime.Scoped);

// 인터페이스로 등록 (권장)
builder.Register<IScoreService, ScoreService>(Lifetime.Scoped);

// 여러 인터페이스로 노출
builder.Register<ScoreService>(Lifetime.Scoped)
    .As<IScoreService>()
    .As<IScoreReader>();

// 구현한 모든 인터페이스로 자동 등록
builder.Register<ScoreService>(Lifetime.Scoped)
    .AsImplementedInterfaces();

// 인터페이스 + 구체 타입 둘 다 Resolve 가능
builder.Register<ScoreService>(Lifetime.Scoped)
    .AsImplementedInterfaces()
    .AsSelf();
```

### 인스턴스 등록

이미 생성된 객체를 등록한다. 항상 Singleton이다.

```csharp
builder.RegisterInstance(gameConfig);
```

### MonoBehaviour 등록

```csharp
// Inspector에서 할당한 참조 등록
[SerializeField] private PlayerView _playerView;

protected override void Configure(IContainerBuilder builder)
{
    // 씬에 이미 존재하는 컴포넌트 (SerializeField)
    builder.RegisterComponent(_playerView);

    // 씬 Hierarchy에서 자동 탐색
    builder.RegisterComponentInHierarchy<PlayerView>();

    // Prefab에서 인스턴스 생성
    builder.RegisterComponentInNewPrefab(_enemyPrefab, Lifetime.Scoped);

    // 새 GameObject에 컴포넌트 생성
    builder.RegisterComponentOnNewGameObject<AudioListener>(
        Lifetime.Singleton, "AudioListener");
}
```

**MonoBehaviour 등록 선택 기준:**

| 메서드 | 사용 시점 |
|---|---|
| `RegisterComponent` | Inspector에서 직접 할당한 참조 |
| `RegisterComponentInHierarchy` | 씬에 이미 배치된 컴포넌트 자동 탐색 |
| `RegisterComponentInNewPrefab` | Prefab 인스턴스 생성이 필요할 때 |
| `RegisterComponentOnNewGameObject` | 빈 GameObject에 컴포넌트 추가 |

### 팩토리 등록

런타임에 동적으로 인스턴스를 생성해야 할 때 사용한다.

```csharp
// 단순 팩토리
builder.RegisterFactory<Bullet>(() => new Bullet());

// 파라미터가 있는 팩토리
builder.RegisterFactory<int, Enemy>(id => new Enemy(id));

// 컨테이너 의존성이 필요한 팩토리
builder.RegisterFactory<int, Enemy>(container =>
{
    var config = container.Resolve<EnemyConfig>();
    return id => new Enemy(id, config);
}, Lifetime.Scoped);
```

> 팩토리가 반환하는 객체의 수명은 VContainer가 관리하지 않는다.
> `IDisposable` 구현체를 팩토리로 생성할 경우 수동으로 Dispose해야 한다.

---

## EntryPoint 패턴

### 순수 C# EntryPoint

MonoBehaviour 대신 순수 C# 클래스로 게임 로직의 진입점을 만든다.

```csharp
public class GamePresenter : IStartable, ITickable, IDisposable
{
    private readonly IScoreService _scoreService;
    private readonly PlayerView _playerView;

    public GamePresenter(IScoreService scoreService, PlayerView playerView)
    {
        _scoreService = scoreService;
        _playerView = playerView;
    }

    void IStartable.Start()
    {
        // MonoBehaviour.Start()와 같은 타이밍
        _playerView.OnScoreChanged += _scoreService.UpdateScore;
    }

    void ITickable.Tick()
    {
        // MonoBehaviour.Update()와 같은 타이밍
    }

    void IDisposable.Dispose()
    {
        _playerView.OnScoreChanged -= _scoreService.UpdateScore;
    }
}
```

등록:

```csharp
builder.RegisterEntryPoint<GamePresenter>();
```

### 사용 가능한 라이프사이클 인터페이스

| 인터페이스 | 타이밍 | 용도 |
|---|---|---|
| `IInitializable` | 컨테이너 빌드 직후 | 초기 설정 |
| `IPostInitializable` | `IInitializable` 이후 | 2차 초기화 |
| `IStartable` | `MonoBehaviour.Start()` | 일반 시작 로직 |
| `IAsyncStartable` | `IStartable`과 동일 | 비동기 시작 로직 |
| `IFixedTickable` | `FixedUpdate()` | 물리 연산 |
| `IPostFixedTickable` | `FixedUpdate()` 이후 | 물리 후처리 |
| `ITickable` | `Update()` | 프레임 로직 |
| `IPostTickable` | `Update()` 이후 | 프레임 후처리 |
| `ILateTickable` | `LateUpdate()` | 카메라 추적 등 |
| `IPostLateTickable` | `LateUpdate()` 이후 | 최종 후처리 |
| `IDisposable` | 컨테이너 파괴 시 | 정리 (Singleton/Scoped) |

**원칙:**
- 하나의 클래스에 필요한 인터페이스만 구현한다
- `IDisposable`은 이벤트 구독 해제가 있을 때 반드시 구현한다
- 비동기 초기화는 `IAsyncStartable`을 사용한다

### 예외 처리

EntryPoint에서 발생한 예외는 기본적으로 `Debug.LogException`으로 출력된다.
커스텀 처리가 필요하면:

```csharp
builder.RegisterEntryPointExceptionHandler(ex =>
{
    // 커스텀 예외 처리
    Debug.LogError($"EntryPoint 예외: {ex.Message}");
});
```

---

## 씬 간 스코프 관리

### 부모 LifetimeScope 지정 (Inspector)

자식 LifetimeScope의 Inspector에서 `Parent` 필드에 부모 타입을 지정한다.
부모가 같은 씬에 없으면 에러가 발생하므로, Additive 씬 로딩 시에는 코드로 지정한다.

### 동적 부모 지정 (Additive Scene)

```csharp
// 부모 스코프에서 자식 씬 로드
using (LifetimeScope.EnqueueParent(this))
{
    await SceneManager.LoadSceneAsync("GameScene", LoadSceneMode.Additive);
}
```

### 씬 로드 시 추가 등록

```csharp
using (LifetimeScope.EnqueueParent(this))
using (LifetimeScope.Enqueue(builder =>
{
    // 자식 씬의 LifetimeScope에 추가 등록
    builder.RegisterInstance(levelData);
}))
{
    await SceneManager.LoadSceneAsync("GameScene", LoadSceneMode.Additive);
}
```

### 기존 스코프 검색

```csharp
var rootScope = LifetimeScope.Find<RootLifetimeScope>();
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `container.Resolve<T>()` 직접 호출 (서비스 로케이터) | 생성자 주입 |
| MonoBehaviour에서 `[Inject]` 남용 | `RegisterComponent`로 등록 후 생성자 주입 가능한 클래스에서 사용 |
| 모든 것을 `Singleton`으로 등록 | `Scoped` 기본, 필요 시에만 `Singleton` |
| LifetimeScope 하나에 모든 등록 | 씬 단위로 분리, 계층 구조 활용 |
| 팩토리 반환 객체의 Dispose 누락 | 팩토리 생성 객체는 수동 Dispose 관리 |
| `Configure`에서 복잡한 로직 실행 | `Configure`는 등록만, 로직은 EntryPoint에서 |
| `Transient`로 상태 있는 서비스 등록 | 상태가 있으면 `Scoped` 또는 `Singleton` |
| 순환 의존성 | 인터페이스 분리 또는 팩토리 패턴으로 해결 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| MVP 패턴에서 VContainer 연동, Presenter 등록 패턴, View-Presenter 바인딩 | `references/mvp-integration.md` |
| 씬 전환 시 데이터 전달, 동적 자식 스코프, Additive Scene 로딩 패턴 | `references/scene-scoping.md` |
