# 씬 전환 및 스코프 관리

## 스코프 계층 설계

### Root Scope (앱 전역)

`DontDestroyOnLoad` 씬에 배치하여 앱 전체에서 공유하는 서비스를 등록한다.

```csharp
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 앱 전역 서비스
        builder.Register<ISaveManager, SaveManager>(Lifetime.Singleton);
        builder.Register<IAudioService, AudioService>(Lifetime.Singleton);
        builder.Register<INetworkClient, NetworkClient>(Lifetime.Singleton);
    }
}
```

### Scene Scope (씬별)

각 씬에 배치하여 씬 내에서만 필요한 의존성을 등록한다.
씬이 언로드되면 Scoped 인스턴스는 자동으로 Dispose된다.

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 게임플레이에서만 필요한 서비스
        builder.Register<IGameStateModel, GameStateModel>(Lifetime.Scoped);
        builder.Register<IEnemySpawner, EnemySpawner>(Lifetime.Scoped);

        builder.RegisterEntryPoint<GamePresenter>();
    }
}
```

---

## Additive Scene 로딩

### EnqueueParent로 부모 지정

Additive로 로드하는 씬의 LifetimeScope에 부모를 동적으로 지정한다.

```csharp
public class GameLifetimeScope : LifetimeScope
{
    public async UniTask LoadUIScene()
    {
        using (LifetimeScope.EnqueueParent(this))
        {
            await SceneManager.LoadSceneAsync("UIScene", LoadSceneMode.Additive);
        }
        // UIScene의 LifetimeScope는 자동으로 this의 자식이 된다
    }
}
```

### 추가 등록과 함께 로드

로드되는 씬의 LifetimeScope에 런타임 데이터를 추가로 등록한다.

```csharp
public async UniTask LoadLevel(LevelData levelData)
{
    using (LifetimeScope.EnqueueParent(this))
    using (LifetimeScope.Enqueue(builder =>
    {
        builder.RegisterInstance(levelData);
        builder.Register<ILevelController, LevelController>(Lifetime.Scoped);
    }))
    {
        await SceneManager.LoadSceneAsync("LevelScene", LoadSceneMode.Additive);
    }
}
```

---

## 씬 전환 패턴

### 단일 씬 전환 (비Additive)

Root Scope가 `DontDestroyOnLoad`에 있으면, 씬 전환 후에도 Root에 등록된 서비스를 사용할 수 있다.

```csharp
// SceneTransitionService는 RootLifetimeScope에 등록
public class SceneTransitionService
{
    public async UniTask TransitionTo(string sceneName)
    {
        await SceneManager.LoadSceneAsync(sceneName);
        // 새 씬의 LifetimeScope는 Inspector에서 Parent Type으로
        // RootLifetimeScope를 지정해 둔다
    }
}
```

### 기존 스코프 검색

이미 로드된 LifetimeScope를 타입으로 검색한다.

```csharp
var rootScope = LifetimeScope.Find<RootLifetimeScope>();
```

---

## 자식 스코프 수동 생성

씬 로딩 없이 코드에서 자식 스코프를 직접 생성할 수 있다.
특정 구간(라운드, 페이즈 등)에서 임시 스코프가 필요할 때 사용한다.

```csharp
public class RoundManager : IDisposable
{
    private readonly LifetimeScope _parentScope;
    private LifetimeScope _roundScope;

    public RoundManager(LifetimeScope parentScope)
    {
        _parentScope = parentScope;
    }

    public void StartRound(RoundData data)
    {
        _roundScope = _parentScope.CreateChild(builder =>
        {
            builder.RegisterInstance(data);
            builder.RegisterEntryPoint<RoundPresenter>();
        });
    }

    public void EndRound()
    {
        _roundScope?.Dispose();
        _roundScope = null;
    }

    void IDisposable.Dispose() => EndRound();
}
```

---

## Dispose 주의사항

### Scoped/Singleton + IDisposable

`IDisposable`을 구현한 Scoped 또는 Singleton 등록 객체는 LifetimeScope 파괴 시 자동으로 `Dispose()`가 호출된다.

### MonoBehaviour의 Dispose

`RegisterComponent`로 등록한 MonoBehaviour는 LifetimeScope가 파괴되어도 자동으로 Destroy되지 않는다.
MonoBehaviour를 LifetimeScope의 자식 Transform으로 배치하거나, `IDisposable`을 구현하여 수동으로 정리한다.

### Transient의 Dispose

`Transient`로 등록한 `IDisposable` 구현체는 컨테이너가 관리하지 않으므로 직접 Dispose해야 한다.
가능하면 `Scoped`로 변경하는 것을 권장한다.
