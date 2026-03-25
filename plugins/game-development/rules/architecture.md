# Architecture Guide (DDD-Lite)

> 이 가이드는 Claude Code가 코드 생성 및 설계 시 반드시 준수해야 할 아키텍처 규칙입니다.
> 목표는 **테스트 가능성**과 **개발 속도** 확보이며, DDD는 그 수단입니다. 오버엔지니어링을 경계합니다.

---

## 1. 레이어 구성

2개의 레이어로 구성하며, **.asmdef로 Assembly를 분리**해 의존 방향을 컴파일 타임에 강제합니다.

```
┌─────────────────────────────────────┐
│          Presentation Layer         │
│  (MonoBehaviour, View, Presenter,   │
│   Infrastructure, VContainer 설정)  │
│                                     │
│  Assembly: {Project}.Presentation   │
└────────────────┬────────────────────┘
                 │ 단방향 의존
┌────────────────▼────────────────────┐
│             Core Layer              │
│  (Domain + Application)             │
│  순수 C# — Unity 런타임 미의존       │
│                                     │
│  Assembly: {Project}.Core           │
└─────────────────────────────────────┘
```

**의존 규칙**
- Presentation → Core (O)
- Core → Presentation (X) — 절대 금지
- Core는 Unity 런타임 타입(`MonoBehaviour`, `GameObject` 등) 참조 금지
- 단, **정적 유틸(`Mathf`, `Vector3` 등)** 과 **순수 데이터 타입**은 EditMode 테스트가 가능하다면 Core에서 사용 가능
- C#과 Unity 양쪽에서 동일 기능을 제공하는 경우 **Unity 타입을 우선 사용**한다. (예: `System.Random` 대신 `UnityEngine.Random`, `System.Math` 대신 `Mathf`)

---

## 2. 폴더 구조

```
Assets/
  Scripts/
    Core/                      # Core Assembly
      Domain/
        {Entity}.cs
        {ValueObject}.cs
        I{Repository}.cs       # 인터페이스만 선언
      Application/
        {UseCase}UseCase.cs
    Presentation/              # Presentation Assembly
      View/
        {Name}View.cs          # 인게임 오브젝트
        {Name}UIView.cs        # UI 패널/팝업
      Presenter/
        {Name}Presenter.cs
        {Name}UIPresenter.cs
      Infrastructure/
        {Repository}Impl.cs    # Repository 구현체
      Installer/
        {Scene}Installer.cs    # VContainer 등록
```

> 파일 수가 많아져 같은 폴더 내 관리가 어려워지는 시점에 Feature 단위 하위 폴더를 추가한다. 처음부터 Feature 폴더를 만들지 않는다.

---

## 3. Core Layer

### 3-1. Domain

필요한 것만 만든다. 아래 개념을 **필요할 때만** 도입한다.

**Entity** — 고유 ID를 가지며 상태가 변하는 객체
```csharp
public class Player
{
    public int Id { get; }
    private int _health;

    public Player(int id, int health)
    {
        Id = id;
        _health = health;
    }

    public void TakeDamage(int amount)
    {
        if (amount <= 0) return;
        _health = Mathf.Max(0, _health - amount); // 정적 유틸 허용
    }

    public bool IsAlive => _health > 0;
}
```

**Value Object** — 불변, 동등성은 값으로 비교
```csharp
public record Position(float X, float Y);
```

**Repository 인터페이스** — Core에는 인터페이스만, 구현은 Presentation/Infrastructure에
```csharp
public interface IPlayerRepository
{
    Player Find(int id);
    void Save(Player player);
}
```

### 3-2. Application (UseCase)

- UseCase 클래스 1개 = 퍼블릭 메서드 1개 (`Execute`)
- 입력/출력이 복잡하면 별도 record로 정의
- 비즈니스 흐름만 기술, UI/인프라 관심사 금지

```csharp
public class AttackEnemyUseCase
{
    private readonly IEnemyRepository _enemyRepository;
    private readonly IPlayerRepository _playerRepository;

    public AttackEnemyUseCase(
        IEnemyRepository enemyRepository,
        IPlayerRepository playerRepository)
    {
        _enemyRepository = enemyRepository;
        _playerRepository = playerRepository;
    }

    public Result Execute(int playerId, int enemyId)
    {
        var player = _playerRepository.Find(playerId);
        if (player == null) return Result.Fail("플레이어를 찾을 수 없습니다.");

        var enemy = _enemyRepository.Find(enemyId);
        if (enemy == null) return Result.Fail("적을 찾을 수 없습니다.");

        enemy.TakeDamage(player.AttackPower);
        _enemyRepository.Save(enemy);

        return Result.Ok();
    }
}
```

---

## 4. Presentation Layer

### 4-1. MVP 패턴

- **View** — MonoBehaviour. UI 이벤트 발행, 시각적 상태 반영만 담당. 로직 금지.
- **View 인터페이스** — View마다 대응하는 인터페이스를 정의하고, Presenter는 인터페이스에만 의존한다. (테스트 시 Fake View 주입 가능)
- **Presenter** — View 인터페이스와 UseCase를 연결. 순수 C# (테스트 용이).

**네이밍 구분**

| 대상 | 접미사 | 예시 |
|---|---|---|
| 인게임 오브젝트 (MonoBehaviour) | `View` / `Presenter` | `CellView`, `EnemyPresenter` |
| UI 패널 / 팝업 (MonoBehaviour) | `UIView` / `UIPresenter` | `MenuUIView`, `SettingUIPresenter` |
| View 인터페이스 | `I{Name}View` / `I{Name}UIView` | `ICellView`, `IMenuUIView` |

```csharp
// View 인터페이스 (Core 또는 Presentation 상단에 정의)
public interface IBattleUIView
{
    IObservable<Unit> OnAttackClicked { get; }
    void UpdateEnemyHp(int hp);
}

// View 구현체
public class BattleUIView : MonoBehaviour, IBattleUIView
{
    [SerializeField] private Button _attackButton;

    public IObservable<Unit> OnAttackClicked => _attackButton.OnClickAsObservable();

    public void UpdateEnemyHp(int hp) { /* UI 갱신 */ }
}

// Presenter — 인터페이스에만 의존
public class BattleUIPresenter : IDisposable
{
    private readonly IBattleUIView _view;
    private readonly AttackEnemyUseCase _attackUseCase;
    private readonly CompositeDisposable _disposables = new();

    public BattleUIPresenter(IBattleUIView view, AttackEnemyUseCase attackUseCase)
    {
        _view = view;
        _attackUseCase = attackUseCase;

        _view.OnAttackClicked
            .Subscribe(_ => OnAttack())
            .AddTo(_disposables);
    }

    private void OnAttack()
    {
        var result = _attackUseCase.Execute(playerId: 1, enemyId: 1);
        if (!result.IsSuccess) return;
        // TODO: 결과 반영
    }

    public void Dispose() => _disposables.Dispose();
}
```

### 4-2. Infrastructure

- Repository 인터페이스의 구현체를 여기에 배치
- 외부 의존(파일, 네트워크, PlayerPrefs 등)은 Infrastructure에만 격리

```csharp
public class PlayerRepositoryImpl : IPlayerRepository
{
    private readonly Dictionary<int, Player> _cache = new();

    public Player Find(int id) => _cache.GetValueOrDefault(id);
    public void Save(Player player) => _cache[player.Id] = player;
}
```

### 4-3. VContainer 등록

```csharp
public class BattleInstaller : LifetimeScope
{
    [SerializeField] private BattleUIView _battleUIView;

    protected override void Configure(IContainerBuilder builder)
    {
        // Infrastructure
        builder.Register<IPlayerRepository, PlayerRepositoryImpl>(Lifetime.Singleton);
        builder.Register<IEnemyRepository, EnemyRepositoryImpl>(Lifetime.Singleton);

        // UseCase
        builder.Register<AttackEnemyUseCase>(Lifetime.Singleton);

        // Presenter — View는 인터페이스로 등록
        builder.RegisterInstance<IBattleUIView>(_battleUIView);
        builder.Register<BattleUIPresenter>(Lifetime.Singleton);
    }
}
```

---

## 5. 이벤트 / 메시지 통신

### 레이어 간 통신 원칙

| 상황 | 방식 |
|---|---|
| Presenter → UseCase 호출 | 직접 메서드 호출 |
| 도메인 상태 변화 알림 | R3 `ReactiveProperty` |
| 레이어 간 느슨한 결합 이벤트 | MessagePipe (pub/sub) |
| View 내부 UI 이벤트 | R3 Observable |

**R3 — 상태 바인딩**
```csharp
// Domain 또는 Application
public class GameState
{
    public ReactiveProperty<int> Score { get; } = new(0);
}

// Presenter에서 구독
_gameState.Score
    .Subscribe(score => _view.UpdateScore(score))
    .AddTo(_disposables);
```

**MessagePipe — 레이어 간 이벤트**
```csharp
// 이벤트 정의 (Core)
public record EnemyDiedEvent(int EnemyId);

// 발행 (UseCase 또는 Presenter)
_publisher.Publish(new EnemyDiedEvent(enemyId));

// 구독 (다른 Presenter)
_subscriber.Subscribe<EnemyDiedEvent>(e => OnEnemyDied(e.EnemyId));
```

---

## 6. 에러 / 결과 처리

- **도메인/Application** — `Result<T>` 타입으로 성공/실패 표현, Exception 사용 금지
- **Infrastructure** — 외부 I/O 예외는 `try-catch`로 잡아 `Result.Fail()`로 변환 후 상위 전달

```csharp
public class Result
{
    public bool IsSuccess { get; }
    public string Error { get; }

    private Result(bool isSuccess, string error = null)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Ok() => new(true);
    public static Result Fail(string error) => new(false, error);
}

public class Result<T> : Result
{
    public T Value { get; }

    private Result(T value) : base(true) => Value = value;
    private Result(string error) : base(false, error) { }

    public static Result<T> Ok(T value) => new(value);
    public new static Result<T> Fail(string error) => new(error);
}
```

---

## 7. 테스트 전략

| 레이어 | 테스트 모드 | 대상 |
|---|---|---|
| Core (Domain + Application) | **EditMode** | UseCase, Entity, Value Object |
| Presentation (Presenter) | EditMode (순수 C# Presenter) | Presenter 로직 |
| Presentation (View, Infrastructure) | PlayMode | UI 연동, 실제 저장소 |

- Core는 Unity 런타임 미의존이므로 **EditMode에서 빠르게 테스트**한다.
- Repository는 테스트용 In-memory 구현체(Fake)를 사용한다.
- Presenter 테스트 시 View 인터페이스의 **Fake 구현체**를 주입해 MonoBehaviour 없이 테스트한다.
- Mocking은 **NSubstitute**를 사용한다.

```csharp
// Fake View — EditMode 테스트용
public class FakeBattleUIView : IBattleUIView
{
    private readonly Subject<Unit> _attackSubject = new();
    public IObservable<Unit> OnAttackClicked => _attackSubject;
    public int LastHp { get; private set; }

    public void UpdateEnemyHp(int hp) => LastHp = hp;
    public void SimulateAttack() => _attackSubject.OnNext(Unit.Default);
}

// Presenter EditMode 테스트
[Test]
public void 공격_버튼_클릭시_적_체력이_감소한다()
{
    var fakeView = new FakeBattleUIView();
    var presenter = new BattleUIPresenter(fakeView, _attackUseCase);

    fakeView.SimulateAttack();

    Assert.That(fakeView.LastHp, Is.LessThan(INITIAL_HP));
}
```
