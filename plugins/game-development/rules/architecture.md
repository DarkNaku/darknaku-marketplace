# Architecture Guide (MVP)

> 이 가이드는 Claude Code가 코드 생성 및 설계 시 반드시 준수해야 할 아키텍처 규칙입니다.
> 목표는 **테스트 가능성**과 **개발 속도** 확보이며, MVP는 그 수단입니다. 오버엔지니어링을 경계합니다.

---

## 1. 레이어 구성

2개의 Assembly로 분리하며, **.asmdef로 의존 방향을 컴파일 타임에 강제**합니다.

```
┌─────────────────────────────────────┐
│          Presentation Layer         │
│  (View, Infrastructure, Installer)  │
│  MonoBehaviour 의존 코드             │
│                                     │
│  Assembly: {Project}.Presentation   │
└────────────────┬────────────────────┘
                 │ 단방향 의존
┌────────────────▼────────────────────┐
│             Core Layer              │
│  (Model, Presenter, Interface)      │
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
- C#과 Unity 양쪽에서 동일 기능을 제공하는 경우 **Unity 타입을 우선 사용**한다 (예: `System.Random` 대신 `UnityEngine.Random`)

---

## 2. 폴더 구조

```
Assets/
  Scripts/
    Core/                          # Core Assembly
      Model/
        {Name}Model.cs             # 데이터 + 비즈니스 로직
      Presenter/
        {Name}Presenter.cs         # 인게임 오브젝트용 Presenter
        {Name}UIPresenter.cs       # UI 패널/팝업용 Presenter
      Interface/
        I{Name}View.cs             # 인게임 오브젝트 View 인터페이스
        I{Name}UIView.cs           # UI 패널/팝업 View 인터페이스
        I{Name}Service.cs          # 외부 참조 인터페이스
    Presentation/                  # Presentation Assembly
      View/
        {Name}View.cs              # 인게임 오브젝트 (MonoBehaviour)
        {Name}UIView.cs            # UI 패널/팝업 (MonoBehaviour)
      Infrastructure/
        {Name}ServiceImpl.cs       # 외부 참조 구현체
      Installer/
        {Scene}Installer.cs        # VContainer 등록
```

---

## 3. Model

- 데이터와 비즈니스 로직을 함께 담는다
- 상태 변화는 R3 `ReactiveProperty`로 노출해 Presenter가 구독할 수 있도록 한다
- 외부 의존 없이 순수 C#으로만 작성한다

```csharp
public class PlayerModel
{
    public ReactiveProperty<int> Health { get; } = new(100);
    public ReactiveProperty<int> Score { get; } = new(0);

    private readonly int _maxHealth;

    public PlayerModel(int maxHealth)
    {
        _maxHealth = maxHealth;
        Health.Value = maxHealth;
    }

    public void TakeDamage(int amount)
    {
        if (amount <= 0) throw new ArgumentException("데미지는 0보다 커야 합니다.");
        Health.Value = Mathf.Max(0, Health.Value - amount);
    }

    public void AddScore(int amount)
    {
        if (amount <= 0) throw new ArgumentException("점수는 0보다 커야 합니다.");
        Score.Value += amount;
    }

    public bool IsAlive => Health.Value > 0;
}
```

---

## 4. View

- **View 인터페이스**는 Core에 선언한다 — Presenter가 인터페이스에만 의존하도록 강제
- **View 구현체**는 Presentation에 위치하며, `MonoBehaviour`를 상속한다
- UI 이벤트는 R3 `IObservable`로 노출하고, 시각적 상태 반영 메서드만 포함한다
- 로직 금지 — View는 보여주고 이벤트를 전달하는 역할만 한다

**네이밍 구분**

| 대상 | 접미사 | 예시 |
|---|---|---|
| 인게임 오브젝트 (MonoBehaviour) | `View` / `Presenter` | `PlayerView`, `EnemyPresenter` |
| UI 패널 / 팝업 (MonoBehaviour) | `UIView` / `UIPresenter` | `HudUIView`, `SettingUIPresenter` |
| View 인터페이스 | `I{Name}View` / `I{Name}UIView` | `IPlayerView`, `IHudUIView` |

```csharp
// Core/View/IHudUIView.cs
public interface IHudUIView
{
    IObservable<Unit> OnAttackClicked { get; }
    void UpdateHealth(int health);
    void UpdateScore(int score);
}

// Presentation/View/HudUIView.cs
public class HudUIView : MonoBehaviour, IHudUIView
{
    [SerializeField] private Button _attackButton;
    [SerializeField] private Text _healthText;
    [SerializeField] private Text _scoreText;

    public IObservable<Unit> OnAttackClicked => _attackButton.OnClickAsObservable();

    public void UpdateHealth(int health) => _healthText.text = health.ToString();
    public void UpdateScore(int score) => _scoreText.text = score.ToString();
}
```

---

## 5. Presenter

- View 인터페이스와 Model을 연결하는 역할
- 순수 C# — `MonoBehaviour` 미사용 (EditMode 테스트 가능)
- View 이벤트를 구독해 Model 메서드를 호출하고, Model 상태 변화를 View에 반영한다
- Presenter 간 느슨한 결합이 필요하면 MessagePipe를 사용한다

```csharp
public class HudUIPresenter : IDisposable
{
    private readonly IHudUIView _view;
    private readonly PlayerModel _playerModel;
    private readonly IPublisher<PlayerDiedEvent> _publisher;
    private readonly CompositeDisposable _disposables = new();

    public HudUIPresenter(
        IHudUIView view,
        PlayerModel playerModel,
        IPublisher<PlayerDiedEvent> publisher)
    {
        _view = view;
        _playerModel = playerModel;
        _publisher = publisher;

        // Model 상태 → View 반영
        _playerModel.Health
            .Subscribe(hp => _view.UpdateHealth(hp))
            .AddTo(_disposables);

        _playerModel.Score
            .Subscribe(score => _view.UpdateScore(score))
            .AddTo(_disposables);

        // View 이벤트 → Model 호출
        _view.OnAttackClicked
            .Subscribe(_ => OnAttack())
            .AddTo(_disposables);
    }

    private void OnAttack()
    {
        // 필요한 처리 수행
        if (!_playerModel.IsAlive)
        {
            _publisher.Publish(new PlayerDiedEvent());
        }
    }

    public void Dispose() => _disposables.Dispose();
}
```

---

## 6. 통신 방식

| 상황 | 방식 |
|---|---|
| View 이벤트 → Presenter | R3 `IObservable` (View 인터페이스 통해 구독) |
| Model 상태 변화 → Presenter → View | R3 `ReactiveProperty` 구독 |
| Presenter 간 느슨한 이벤트 | MessagePipe (pub/sub) |

**MessagePipe — Presenter 간 이벤트**
```csharp
// Core에 이벤트 정의
public record PlayerDiedEvent();

// 발행 (Presenter)
_publisher.Publish(new PlayerDiedEvent());

// 구독 (다른 Presenter)
_subscriber.Subscribe<PlayerDiedEvent>(_ => OnPlayerDied())
    .AddTo(_disposables);
```

---

## 7. 외부 참조 인터페이스

외부 의존(파일, 네트워크, PlayerPrefs 등)이 필요한 경우 인터페이스를 Core에 선언하고 구현체는 Presentation/Infrastructure에 배치한다.

```csharp
// Core/Service/ISaveService.cs
public interface ISaveService
{
    void Save(string key, int value);
    int Load(string key, int defaultValue = 0);
}

// Presentation/Infrastructure/PlayerPrefsSaveService.cs
public class PlayerPrefsSaveService : ISaveService
{
    public void Save(string key, int value) => PlayerPrefs.SetInt(key, value);
    public int Load(string key, int defaultValue = 0) => PlayerPrefs.GetInt(key, defaultValue);
}
```

---

## 8. VContainer 등록

```csharp
public class BattleInstaller : LifetimeScope
{
    [SerializeField] private HudUIView _hudUIView;

    protected override void Configure(IContainerBuilder builder)
    {
        // Infrastructure
        builder.Register<ISaveService, PlayerPrefsSaveService>(Lifetime.Singleton);

        // MessagePipe
        var options = builder.RegisterMessagePipe();
        builder.RegisterMessageBroker<PlayerDiedEvent>(options);

        // Model
        builder.Register<PlayerModel>(Lifetime.Singleton);

        // View — 인터페이스로 등록
        builder.RegisterInstance<IHudUIView>(_hudUIView);

        // Presenter
        builder.Register<HudUIPresenter>(Lifetime.Singleton);
    }
}
```

---

## 9. 테스트 전략

| 레이어 | 테스트 모드 | 대상 |
|---|---|---|
| Core (Model) | **EditMode** | 비즈니스 로직, 상태 변화 |
| Core (Presenter) | **EditMode** | View 이벤트 → Model 호출, Model 상태 → View 반영 |
| Presentation (View, Infrastructure) | **PlayMode** | UI 연동, 실제 외부 서비스 |

- Presenter 테스트 시 View 인터페이스의 **Fake 구현체**를 주입해 MonoBehaviour 없이 테스트한다
- 외부 서비스 인터페이스는 **NSubstitute**로 목킹한다

```csharp
// Fake View — EditMode 테스트용
public class FakeHudUIView : IHudUIView
{
    private readonly Subject<Unit> _attackSubject = new();
    public IObservable<Unit> OnAttackClicked => _attackSubject;
    public int LastHealth { get; private set; }
    public int LastScore { get; private set; }

    public void UpdateHealth(int health) => LastHealth = health;
    public void UpdateScore(int score) => LastScore = score;
    public void SimulateAttack() => _attackSubject.OnNext(Unit.Default);
}

// Model EditMode 테스트
[Test]
public void 데미지를_받으면_체력이_감소한다()
{
    var model = new PlayerModel(maxHealth: 100);

    model.TakeDamage(30);

    Assert.That(model.Health.Value, Is.EqualTo(70));
}

// Presenter EditMode 테스트
[Test]
public void 공격_버튼_클릭시_모델_메서드가_호출된다()
{
    var fakeView = new FakeHudUIView();
    var model = new PlayerModel(maxHealth: 100);
    var publisher = GlobalMessagePipe.GetPublisher<PlayerDiedEvent>();
    var presenter = new HudUIPresenter(fakeView, model, publisher);

    model.TakeDamage(30);

    Assert.That(fakeView.LastHealth, Is.EqualTo(70));
}
```