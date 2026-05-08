# VContainer + MessagePipe + R3 통합 아키텍처

## 아키텍처 개요

세 라이브러리의 역할을 명확히 분리하여 사용한다.

```
VContainer (DI)
├── 의존성 등록 및 주입
├── LifetimeScope 계층 관리
└── EntryPoint 수명 관리

MessagePipe (이벤트 브로커)
├── 시스템 간 디커플링 이벤트 전달
├── Request/Response 커맨드 패턴
└── 키 기반 엔티티별 이벤트

R3 (리액티브 상태)
├── ReactiveProperty 상태 바인딩
├── Observable 오퍼레이터 체인
└── 프레임/시간 기반 스트림
```

---

## 전형적인 게임 아키텍처

```
┌─────────────────────────────────────────────────┐
│                  LifetimeScope                   │
│                                                  │
│  ┌──────────┐    MessagePipe    ┌──────────┐    │
│  │  System A │ ──────────────→ │  System B │    │
│  │(Publisher)│  IPublisher<T>   │(Subscriber)│   │
│  └──────────┘                  └──────────┘    │
│                                                  │
│  ┌──────────┐   R3 Property    ┌──────────┐    │
│  │   Model   │ ──────────────→ │ Presenter │    │
│  │(Property) │  Subscribe()    │  (View)   │    │
│  └──────────┘                  └──────────┘    │
│                                                  │
│  ┌──────────┐  Request/Response ┌──────────┐   │
│  │ Presenter │ ──────────────→ │  Handler  │    │
│  │(Requester)│  IRequestHandler │(Executor) │   │
│  └──────────┘                  └──────────┘    │
└─────────────────────────────────────────────────┘
```

---

## 통합 LifetimeScope 설정

```csharp
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private GameView _gameView;
    [SerializeField] private HudView _hudView;

    protected override void Configure(IContainerBuilder builder)
    {
        // === MessagePipe ===
        var options = builder.RegisterMessagePipe();
        builder.RegisterBuildCallback(c =>
            GlobalMessagePipe.SetProvider(c.AsServiceProvider()));

        // 이벤트 메시지 등록
        builder.RegisterMessageBroker<GameStarted>(options);
        builder.RegisterMessageBroker<EnemyDefeated>(options);
        builder.RegisterMessageBroker<ItemPickedUp>(options);
        builder.RegisterMessageBroker<LevelCompleted>(options);

        // Request/Response 등록
        builder.RegisterRequestHandler<
            CalculateDamageRequest, DamageResult, DamageCalculator>(options);

        // === Models (R3 ReactiveProperty 내부 사용) ===
        builder.Register<IPlayerModel, PlayerModel>(Lifetime.Scoped);
        builder.Register<IScoreModel, ScoreModel>(Lifetime.Scoped);
        builder.Register<IInventoryModel, InventoryModel>(Lifetime.Scoped);

        // === Systems (MessagePipe로 이벤트 발행) ===
        builder.Register<ICombatSystem, CombatSystem>(Lifetime.Scoped);
        builder.Register<ISpawnSystem, SpawnSystem>(Lifetime.Scoped);

        // === Views ===
        builder.RegisterComponent(_gameView);
        builder.RegisterComponent(_hudView);

        // === Presenters (EntryPoint) ===
        builder.RegisterEntryPoint<GamePresenter>();
        builder.RegisterEntryPoint<HudPresenter>();
        builder.RegisterEntryPoint<ScorePresenter>();
    }
}
```

---

## Model 계층 (R3)

Model은 R3 `ReactiveProperty`로 상태를 관리한다.

```csharp
public interface IPlayerModel
{
    ReadOnlyReactiveProperty<int> Hp { get; }
    ReadOnlyReactiveProperty<int> MaxHp { get; }
    ReadOnlyReactiveProperty<bool> IsDead { get; }
    void TakeDamage(int amount);
    void Heal(int amount);
}

public class PlayerModel : IPlayerModel, IDisposable
{
    private readonly ReactiveProperty<int> _hp;
    private readonly ReactiveProperty<int> _maxHp;

    public ReadOnlyReactiveProperty<int> Hp => _hp;
    public ReadOnlyReactiveProperty<int> MaxHp => _maxHp;
    public ReadOnlyReactiveProperty<bool> IsDead { get; }

    public PlayerModel(GameConfig config)
    {
        _maxHp = new ReactiveProperty<int>(config.PlayerMaxHp);
        _hp = new ReactiveProperty<int>(config.PlayerMaxHp);
        IsDead = _hp.Select(hp => hp <= 0).ToReadOnlyReactiveProperty();
    }

    public void TakeDamage(int amount)
        => _hp.Value = Math.Max(0, _hp.Value - amount);

    public void Heal(int amount)
        => _hp.Value = Math.Min(_maxHp.Value, _hp.Value + amount);

    public void Dispose()
    {
        _hp.Dispose();
        _maxHp.Dispose();
    }
}
```

---

## System 계층 (MessagePipe 발행)

System은 게임 로직을 처리하고 MessagePipe로 이벤트를 발행한다.

```csharp
public class CombatSystem : ICombatSystem
{
    private readonly IPlayerModel _player;
    private readonly IRequestHandler<CalculateDamageRequest, DamageResult> _damageCalc;
    private readonly IPublisher<EnemyDefeated> _enemyDefeatedPub;

    public CombatSystem(
        IPlayerModel player,
        IRequestHandler<CalculateDamageRequest, DamageResult> damageCalc,
        IPublisher<EnemyDefeated> enemyDefeatedPub)
    {
        _player = player;
        _damageCalc = damageCalc;
        _enemyDefeatedPub = enemyDefeatedPub;
    }

    public void AttackEnemy(Enemy enemy, bool isCrit)
    {
        var result = _damageCalc.Invoke(
            new CalculateDamageRequest(enemy.Defense, 2.0f, isCrit));

        enemy.TakeDamage(result.FinalDamage);

        if (enemy.IsDead)
        {
            _enemyDefeatedPub.Publish(new EnemyDefeated(
                enemy.Id, enemy.ScoreValue, enemy.Position));
        }
    }
}
```

---

## Presenter 계층 (R3 구독 + MessagePipe 구독)

Presenter는 R3로 Model 상태를 View에 바인딩하고,
MessagePipe로 시스템 이벤트를 수신하여 UI를 업데이트한다.

```csharp
public class HudPresenter : IStartable, IDisposable
{
    private readonly IPlayerModel _player;
    private readonly ISubscriber<EnemyDefeated> _enemyDefeatedSub;
    private readonly HudView _view;
    private DisposableBag _r3Disposables;
    private IDisposable _messagePipeDisposable;

    public HudPresenter(
        IPlayerModel player,
        ISubscriber<EnemyDefeated> enemyDefeatedSub,
        HudView view)
    {
        _player = player;
        _enemyDefeatedSub = enemyDefeatedSub;
        _view = view;
    }

    void IStartable.Start()
    {
        // R3: 상태 바인딩
        _player.Hp
            .Subscribe(hp => _view.SetHp(hp))
            .AddTo(ref _r3Disposables);

        Observable.CombineLatest(
            _player.Hp, _player.MaxHp,
            (hp, max) => (float)hp / max)
            .Subscribe(ratio => _view.SetHpRatio(ratio))
            .AddTo(ref _r3Disposables);

        // MessagePipe: 이벤트 수신
        var bag = DisposableBag.CreateBuilder();

        _enemyDefeatedSub
            .Subscribe(msg => _view.ShowKillEffect(msg.Position))
            .AddTo(bag);

        _messagePipeDisposable = bag.Build();
    }

    void IDisposable.Dispose()
    {
        _r3Disposables.Dispose();
        _messagePipeDisposable?.Dispose();
    }
}
```

---

## 시스템 간 통신 패턴

### 단방향 통신 (System → System)

```
CombatSystem ──[EnemyDefeated]──→ SpawnSystem
             ──[EnemyDefeated]──→ ScorePresenter
             ──[EnemyDefeated]──→ EffectSystem
```

발행 한 번으로 여러 시스템이 독립적으로 반응한다.

### 요청-응답 통신

```
CombatSystem ──[CalculateDamageRequest]──→ DamageCalculator
             ←──[DamageResult]──────────┘
```

### 씬 간 통신

부모 LifetimeScope에 메시지 브로커를 등록하면,
자식 씬의 시스템도 동일 브로커를 공유한다.

```csharp
// RootLifetimeScope (DontDestroyOnLoad)
public class RootLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        var options = builder.RegisterMessagePipe();

        // 전역 이벤트 (씬 전환 알림 등)
        builder.RegisterMessageBroker<SceneTransitionEvent>(options);
        builder.RegisterMessageBroker<NetworkEvent>(options);
    }
}

// GameLifetimeScope (게임 씬)
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        var options = builder.RegisterMessagePipe();

        // 씬 내부 이벤트
        builder.RegisterMessageBroker<EnemyDefeated>(options);
        builder.RegisterMessageBroker<ItemPickedUp>(options);

        // RootLifetimeScope의 SceneTransitionEvent는
        // 부모 스코프에서 자동 Resolve된다
    }
}
```
