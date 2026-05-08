# MVP + VContainer 연동 패턴

## 기본 MVP 구조

VContainer에서 MVP 패턴은 다음과 같이 역할을 분리한다:

- **Model**: 순수 C# 클래스. 게임 상태와 비즈니스 로직 담당
- **View**: MonoBehaviour 또는 VisualElement 래퍼. UI 표현만 담당
- **Presenter**: 순수 C# EntryPoint. Model과 View를 중재

```
Model ← Presenter → View
         ↑
    VContainer가 주입
```

---

## Presenter 구현

Presenter는 `IStartable`과 `IDisposable`을 구현하는 순수 C# 클래스다.

```csharp
public class HudPresenter : IStartable, IDisposable
{
    private readonly IGameStateModel _model;
    private readonly HudView _view;

    public HudPresenter(IGameStateModel model, HudView view)
    {
        _model = model;
        _view = view;
    }

    void IStartable.Start()
    {
        // Model → View
        _model.OnScoreChanged += _view.SetScore;
        _model.OnHealthChanged += _view.SetHealthRatio;

        // View → Model
        _view.OnPauseClicked += HandlePause;

        // 초기값 반영
        _view.SetScore(_model.CurrentScore);
        _view.SetHealthRatio(_model.CurrentHealthRatio);
    }

    private void HandlePause() => _model.RequestPause();

    void IDisposable.Dispose()
    {
        _model.OnScoreChanged -= _view.SetScore;
        _model.OnHealthChanged -= _view.SetHealthRatio;
        _view.OnPauseClicked -= HandlePause;
    }
}
```

---

## LifetimeScope 등록

```csharp
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private HudView _hudView;

    protected override void Configure(IContainerBuilder builder)
    {
        // Model
        builder.Register<IGameStateModel, GameStateModel>(Lifetime.Scoped);

        // View (씬에 배치된 MonoBehaviour)
        builder.RegisterComponent(_hudView);

        // Presenter (EntryPoint로 등록 — IStartable/IDisposable 자동 호출)
        builder.RegisterEntryPoint<HudPresenter>();
    }
}
```

**핵심:**
- `RegisterEntryPoint`는 `IStartable.Start()`와 `IDisposable.Dispose()`를 자동으로 호출한다
- Presenter에 별도의 Lifetime을 지정할 필요가 없다 (EntryPoint 기본은 Singleton)
- View는 `RegisterComponent`로 등록하여 Presenter 생성자에 주입된다

---

## 여러 Presenter 등록

하나의 LifetimeScope에서 여러 Presenter를 등록할 수 있다.

```csharp
protected override void Configure(IContainerBuilder builder)
{
    // Models
    builder.Register<IGameStateModel, GameStateModel>(Lifetime.Scoped);
    builder.Register<IInventoryModel, InventoryModel>(Lifetime.Scoped);

    // Views
    builder.RegisterComponent(_hudView);
    builder.RegisterComponent(_inventoryView);

    // Presenters
    builder.RegisterEntryPoint<HudPresenter>();
    builder.RegisterEntryPoint<InventoryPresenter>();
}
```

---

## UI Toolkit View + VContainer

UI Toolkit 기반 View를 VContainer와 연동할 때의 패턴.

```csharp
// View (MonoBehaviour + UIDocument)
public class HudView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;

    private Label _scoreLabel;
    private Button _pauseButton;

    public event System.Action OnPauseClicked;

    private void Awake()
    {
        var root = _uiDocument.rootVisualElement;
        _scoreLabel = root.Q<Label>("score-label");
        _pauseButton = root.Q<Button>("pause-button");
        _pauseButton.clicked += OnPauseButtonClicked;
    }

    public void SetScore(int score) => _scoreLabel.text = score.ToString("N0");

    private void OnPauseButtonClicked() => OnPauseClicked?.Invoke();

    private void OnDestroy()
    {
        _pauseButton.clicked -= OnPauseButtonClicked;
    }
}
```

등록과 Presenter는 MonoBehaviour View와 동일하다. View의 내부 구현만 UI Toolkit으로 바뀐다.

---

## R3 (Reactive Extensions) 연동

R3를 사용하면 이벤트 바인딩을 선언적으로 작성할 수 있다.

```csharp
public class HudPresenter : IStartable, IDisposable
{
    private readonly IGameStateModel _model;
    private readonly HudView _view;
    private readonly CompositeDisposable _disposables = new();

    public HudPresenter(IGameStateModel model, HudView view)
    {
        _model = model;
        _view = view;
    }

    void IStartable.Start()
    {
        _model.Score
            .Subscribe(score => _view.SetScore(score))
            .AddTo(_disposables);

        _model.HealthRatio
            .Subscribe(ratio => _view.SetHealthRatio(ratio))
            .AddTo(_disposables);
    }

    void IDisposable.Dispose()
    {
        _disposables.Dispose();
    }
}
```

이 경우 Model의 프로퍼티는 `ReactiveProperty<T>`로 선언한다:

```csharp
public class GameStateModel : IGameStateModel
{
    public ReactiveProperty<int> Score { get; } = new(0);
    public ReactiveProperty<float> HealthRatio { get; } = new(1f);
}
```
