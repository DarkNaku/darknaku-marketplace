# R3 + VContainer 통합 패턴

## Presenter에서의 구독 관리

VContainer EntryPoint(Presenter)에서 R3를 사용할 때의 표준 패턴.

### 기본 패턴: DisposableBag

```csharp
public class GamePresenter : IStartable, IDisposable
{
    private readonly IGameModel _model;
    private readonly GameView _view;
    private DisposableBag _disposables;

    public GamePresenter(IGameModel model, GameView view)
    {
        _model = model;
        _view = view;
    }

    void IStartable.Start()
    {
        // Model → View (상태 바인딩)
        _model.Score
            .Subscribe(score => _view.SetScore(score))
            .AddTo(ref _disposables);

        _model.Hp
            .Subscribe(hp => _view.SetHp(hp))
            .AddTo(ref _disposables);

        // 파생 상태
        _model.Hp
            .Select(hp => (float)hp / _model.MaxHp)
            .Subscribe(ratio => _view.SetHpRatio(ratio))
            .AddTo(ref _disposables);

        // View → Model (이벤트 전달)
        _view.OnAttackClicked
            .Subscribe(_ => _model.Attack())
            .AddTo(ref _disposables);
    }

    void IDisposable.Dispose()
    {
        _disposables.Dispose();
    }
}
```

### LifetimeScope 등록

```csharp
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private GameView _gameView;

    protected override void Configure(IContainerBuilder builder)
    {
        // Model (ReactiveProperty를 내부적으로 사용)
        builder.Register<IGameModel, GameModel>(Lifetime.Scoped);

        // View
        builder.RegisterComponent(_gameView);

        // Presenter (EntryPoint → IStartable.Start/IDisposable.Dispose 자동 호출)
        builder.RegisterEntryPoint<GamePresenter>();
    }
}
```

---

## Model 설계

### ReactiveProperty 기반 Model

```csharp
public interface IGameModel
{
    ReadOnlyReactiveProperty<int> Score { get; }
    ReadOnlyReactiveProperty<int> Hp { get; }
    int MaxHp { get; }
    void Attack();
    void TakeDamage(int damage);
}

public class GameModel : IGameModel
{
    private readonly ReactiveProperty<int> _score = new(0);
    private readonly ReactiveProperty<int> _hp;

    public ReadOnlyReactiveProperty<int> Score => _score;
    public ReadOnlyReactiveProperty<int> Hp => _hp;
    public int MaxHp { get; }

    public GameModel(GameConfig config)
    {
        MaxHp = config.MaxHp;
        _hp = new ReactiveProperty<int>(MaxHp);
    }

    public void Attack() => _score.Value += 10;

    public void TakeDamage(int damage)
        => _hp.Value = Math.Max(0, _hp.Value - damage);
}
```

### Subject 기반 이벤트

상태가 아닌 일회성 이벤트는 Subject로 발행한다.

```csharp
public interface IGameEvents
{
    Observable<Enemy> OnEnemyDefeated { get; }
    Observable<Item> OnItemPickedUp { get; }
}

public class GameEvents : IGameEvents, IDisposable
{
    private readonly Subject<Enemy> _onEnemyDefeated = new();
    private readonly Subject<Item> _onItemPickedUp = new();

    public Observable<Enemy> OnEnemyDefeated => _onEnemyDefeated;
    public Observable<Item> OnItemPickedUp => _onItemPickedUp;

    public void NotifyEnemyDefeated(Enemy enemy) => _onEnemyDefeated.OnNext(enemy);
    public void NotifyItemPickedUp(Item item) => _onItemPickedUp.OnNext(item);

    public void Dispose()
    {
        _onEnemyDefeated.Dispose();
        _onItemPickedUp.Dispose();
    }
}
```

등록:

```csharp
builder.Register<GameEvents>(Lifetime.Scoped)
    .As<IGameEvents>()
    .AsSelf();  // NotifyXxx 호출용
```

---

## View 설계

### MonoBehaviour View (uGUI)

```csharp
public class GameView : MonoBehaviour
{
    [SerializeField] private Text _scoreText;
    [SerializeField] private Slider _hpSlider;
    [SerializeField] private Button _attackButton;

    // Subject로 이벤트 노출
    private readonly Subject<Unit> _onAttackClicked = new();
    public Observable<Unit> OnAttackClicked => _onAttackClicked;

    private void Awake()
    {
        _attackButton.OnClickAsObservable()
            .Subscribe(_ => _onAttackClicked.OnNext(Unit.Default))
            .AddTo(this);
    }

    public void SetScore(int score) => _scoreText.text = score.ToString("N0");
    public void SetHp(int hp) => _scoreText.text = hp.ToString();
    public void SetHpRatio(float ratio) => _hpSlider.value = ratio;

    private void OnDestroy() => _onAttackClicked.Dispose();
}
```

### MonoBehaviour View (UI Toolkit)

```csharp
public class GameView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;

    private Label _scoreLabel;
    private VisualElement _hpBarFill;
    private Button _attackButton;

    private readonly Subject<Unit> _onAttackClicked = new();
    public Observable<Unit> OnAttackClicked => _onAttackClicked;

    private void Awake()
    {
        var root = _uiDocument.rootVisualElement;
        _scoreLabel = root.Q<Label>("score-label");
        _hpBarFill = root.Q<VisualElement>("hp-bar-fill");
        _attackButton = root.Q<Button>("attack-button");

        _attackButton.clicked += HandleAttackClicked;
    }

    public void SetScore(int score) => _scoreLabel.text = score.ToString("N0");
    public void SetHpRatio(float ratio)
        => _hpBarFill.style.width = Length.Percent(ratio * 100f);

    private void HandleAttackClicked() => _onAttackClicked.OnNext(Unit.Default);

    private void OnDestroy()
    {
        _attackButton.clicked -= HandleAttackClicked;
        _onAttackClicked.Dispose();
    }
}
```

---

## 여러 Presenter 간 통신

Presenter 간 직접 참조는 금지한다. 공유 Model 또는 이벤트 버스를 통해 통신한다.

```csharp
// GamePresenter: 적 처치 시 이벤트 발행
public class GamePresenter : IStartable, IDisposable
{
    private readonly GameEvents _events;
    private DisposableBag _disposables;

    void IStartable.Start()
    {
        // 적 처치 로직에서
        _events.NotifyEnemyDefeated(enemy);
    }

    void IDisposable.Dispose() => _disposables.Dispose();
}

// ScorePresenter: 적 처치 이벤트 구독
public class ScorePresenter : IStartable, IDisposable
{
    private readonly IGameEvents _events;
    private readonly IScoreModel _model;
    private DisposableBag _disposables;

    void IStartable.Start()
    {
        _events.OnEnemyDefeated
            .Subscribe(enemy => _model.AddScore(enemy.ScoreValue))
            .AddTo(ref _disposables);
    }

    void IDisposable.Dispose() => _disposables.Dispose();
}
```

---

## CombineLatest로 복합 UI 바인딩

여러 ReactiveProperty를 결합하여 하나의 UI 상태를 만든다.

```csharp
void IStartable.Start()
{
    // HP + MaxHP → 비율 + 색상
    Observable.CombineLatest(
        _model.Hp,
        _model.MaxHp,
        (hp, max) => new { Ratio = (float)hp / max, IsLow = hp < max * 0.3f })
        .Subscribe(state =>
        {
            _view.SetHpRatio(state.Ratio);
            _view.SetHpWarning(state.IsLow);
        })
        .AddTo(ref _disposables);

    // 여러 조건 → 버튼 활성화
    Observable.CombineLatest(
        _model.Hp.Select(hp => hp > 0),
        _model.Cooldown.Select(cd => cd <= 0f),
        (alive, ready) => alive && ready)
        .DistinctUntilChanged()
        .Subscribe(canAttack => _view.SetAttackEnabled(canAttack))
        .AddTo(ref _disposables);
}
```
