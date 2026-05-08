---
name: unity-messagepipe
description: >
  MessagePipe(메시지 브로커)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 Pub/Sub 패턴, Request/Response 패턴, VContainer 연동 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - MessagePipe, IPublisher, ISubscriber 언급 시
  - IRequestHandler, IAsyncRequestHandler 관련 작업
  - Pub/Sub, 메시지 브로커, 이벤트 버스 관련 작업
  - MessageHandlerFilter, GlobalMessagePipe 관련 작업
  - 컴포넌트 간 디커플링된 통신이 필요한 설계

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - MessagePipe를 사용하지 않는 프로젝트
  - 단순 C# event나 Action 콜백으로 충분한 1:1 통신
  - R3 ReactiveProperty로 충분한 상태 바인딩 (unity-r3 스킬 참조)
user-invocable: false
---

# Unity MessagePipe Skill

## 핵심 원칙

> **MessagePipe는 디커플링된 메시지 통신을 위한 라이브러리다.**
> 발행자와 구독자가 서로를 직접 참조하지 않는 Pub/Sub 패턴을 제공한다.
> 1:1 상태 바인딩에는 R3 ReactiveProperty를, 1:N 이벤트 브로드캐스트에는 MessagePipe를 사용한다.

> **구독은 반드시 해제한다.**
> 모든 `Subscribe` 호출의 반환값(`IDisposable`)을 `DisposableBag`으로 관리한다.

공식 저장소: https://github.com/Cysharp/MessagePipe

---

## MessagePipe vs R3 vs C# event 선택 기준

| 상황 | 선택 | 이유 |
|---|---|---|
| 상태 변경 관찰 (HP, 점수 등) | R3 `ReactiveProperty` | 현재 값 보유, 중복 무시, 구독 시 즉시 값 수신 |
| 값 변환/필터링/결합이 필요한 이벤트 스트림 | R3 `Observable` | LINQ 오퍼레이터 체인 |
| 1:N 디커플링 이벤트 (시스템 간 통신) | MessagePipe `IPublisher/ISubscriber` | 발행자/구독자 분리, DI 기반 |
| Request/Response (커맨드 패턴) | MessagePipe `IRequestHandler` | 요청-응답 쌍, 핸들러 분리 |
| 1:1 직접 콜백 (같은 클래스 내부) | C# `event` / `Action` | 가장 단순, 오버헤드 없음 |

---

## VContainer 연동 설정

### LifetimeScope에서 등록

```csharp
using MessagePipe;
using VContainer;
using VContainer.Unity;

public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // MessagePipe 옵션 등록
        var options = builder.RegisterMessagePipe();

        // GlobalMessagePipe 활성화 (선택, 디버그 윈도우용)
        builder.RegisterBuildCallback(c =>
            GlobalMessagePipe.SetProvider(c.AsServiceProvider()));

        // 메시지 브로커 등록
        builder.RegisterMessageBroker<GameStarted>(options);
        builder.RegisterMessageBroker<EnemyDefeated>(options);
        builder.RegisterMessageBroker<ScoreChanged>(options);

        // 키 기반 메시지 브로커 (선택)
        builder.RegisterMessageBroker<int, DamageEvent>(options);

        // Request/Response 핸들러 등록
        builder.RegisterRequestHandler<SaveRequest, SaveResult, SaveHandler>(options);
    }
}
```

### 메시지 타입 정의

메시지는 불변 구조체로 정의한다. 가볍고 할당이 없다.

```csharp
// 데이터 없는 이벤트
public readonly struct GameStarted { }
public readonly struct GamePaused { }

// 데이터가 있는 이벤트
public readonly struct EnemyDefeated
{
    public readonly int EnemyId;
    public readonly int ScoreValue;
    public readonly Vector3 Position;

    public EnemyDefeated(int enemyId, int scoreValue, Vector3 position)
    {
        EnemyId = enemyId;
        ScoreValue = scoreValue;
        Position = position;
    }
}

public readonly struct DamageEvent
{
    public readonly int Amount;
    public readonly DamageType Type;

    public DamageEvent(int amount, DamageType type)
    {
        Amount = amount;
        Type = type;
    }
}
```

---

## Pub/Sub 패턴

### 발행 (Publish)

```csharp
public class EnemySystem : ITickable
{
    private readonly IPublisher<EnemyDefeated> _enemyDefeatedPublisher;

    public EnemySystem(IPublisher<EnemyDefeated> enemyDefeatedPublisher)
    {
        _enemyDefeatedPublisher = enemyDefeatedPublisher;
    }

    private void OnEnemyKilled(Enemy enemy)
    {
        _enemyDefeatedPublisher.Publish(new EnemyDefeated(
            enemy.Id, enemy.ScoreValue, enemy.Position));
    }
}
```

### 구독 (Subscribe)

```csharp
public class ScorePresenter : IStartable, IDisposable
{
    private readonly ISubscriber<EnemyDefeated> _enemyDefeatedSubscriber;
    private readonly ScoreView _view;
    private IDisposable _disposable;

    public ScorePresenter(
        ISubscriber<EnemyDefeated> enemyDefeatedSubscriber,
        ScoreView view)
    {
        _enemyDefeatedSubscriber = enemyDefeatedSubscriber;
        _view = view;
    }

    void IStartable.Start()
    {
        var bag = DisposableBag.CreateBuilder();

        _enemyDefeatedSubscriber
            .Subscribe(msg => _view.AddScore(msg.ScoreValue))
            .AddTo(bag);

        _disposable = bag.Build();
    }

    void IDisposable.Dispose() => _disposable?.Dispose();
}
```

### 여러 메시지 구독

```csharp
public class GameUIPresenter : IStartable, IDisposable
{
    private readonly ISubscriber<GameStarted> _gameStartedSub;
    private readonly ISubscriber<GamePaused> _gamePausedSub;
    private readonly ISubscriber<EnemyDefeated> _enemyDefeatedSub;
    private IDisposable _disposable;

    public GameUIPresenter(
        ISubscriber<GameStarted> gameStartedSub,
        ISubscriber<GamePaused> gamePausedSub,
        ISubscriber<EnemyDefeated> enemyDefeatedSub)
    {
        _gameStartedSub = gameStartedSub;
        _gamePausedSub = gamePausedSub;
        _enemyDefeatedSub = enemyDefeatedSub;
    }

    void IStartable.Start()
    {
        var bag = DisposableBag.CreateBuilder();

        _gameStartedSub.Subscribe(_ => ShowGameUI()).AddTo(bag);
        _gamePausedSub.Subscribe(_ => ShowPauseMenu()).AddTo(bag);
        _enemyDefeatedSub.Subscribe(msg => ShowKillEffect(msg.Position)).AddTo(bag);

        _disposable = bag.Build();
    }

    void IDisposable.Dispose() => _disposable?.Dispose();
}
```

---

## 키 기반 Pub/Sub

엔티티 ID 등으로 메시지를 분류하여 특정 대상만 수신한다.

```csharp
// 등록
builder.RegisterMessageBroker<int, DamageEvent>(options);

// 발행: 특정 엔티티에게 데미지
public class CombatSystem
{
    private readonly IPublisher<int, DamageEvent> _damagePublisher;

    public void ApplyDamage(int targetId, int amount, DamageType type)
    {
        _damagePublisher.Publish(targetId, new DamageEvent(amount, type));
    }
}

// 구독: 자신의 ID에 해당하는 데미지만 수신
public class HealthComponent : MonoBehaviour
{
    [Inject] private ISubscriber<int, DamageEvent> _damageSub;

    private IDisposable _subscription;
    private int _entityId;

    void Start()
    {
        _subscription = _damageSub.Subscribe(_entityId, msg =>
        {
            TakeDamage(msg.Amount, msg.Type);
        });
    }

    void OnDestroy() => _subscription?.Dispose();
}
```

---

## 비동기 Pub/Sub

구독자의 비동기 처리가 완료될 때까지 대기해야 할 때 사용한다.

```csharp
// 등록
builder.RegisterMessageBroker<SaveRequest>(options);  // async도 동일 등록

// 발행 (비동기 대기)
public class SaveSystem
{
    private readonly IAsyncPublisher<SaveRequest> _savePublisher;

    public async UniTask SaveAll()
    {
        // 모든 구독자의 처리가 완료될 때까지 대기
        await _savePublisher.PublishAsync(new SaveRequest(), default);
    }

    public void SaveFireAndForget()
    {
        // 대기하지 않고 발행
        _savePublisher.Publish(new SaveRequest());
    }
}

// 구독 (비동기 핸들러)
public class InventorySaver : IStartable, IDisposable
{
    private readonly IAsyncSubscriber<SaveRequest> _saveSub;
    private IDisposable _disposable;

    void IStartable.Start()
    {
        var bag = DisposableBag.CreateBuilder();
        _saveSub.Subscribe(async (msg, ct) =>
        {
            await SaveInventoryAsync(ct);
        }).AddTo(bag);
        _disposable = bag.Build();
    }

    void IDisposable.Dispose() => _disposable?.Dispose();
}
```

---

## Request/Response 패턴

요청을 보내고 응답을 받는 Mediator 패턴. 커맨드 처리에 적합하다.

### 단일 핸들러

```csharp
// 요청/응답 메시지 정의
public readonly struct SaveRequest
{
    public readonly string SlotName;
    public SaveRequest(string slotName) => SlotName = slotName;
}

public readonly struct SaveResult
{
    public readonly bool Success;
    public readonly string Message;
    public SaveResult(bool success, string message)
    {
        Success = success;
        Message = message;
    }
}

// 핸들러 구현
public class SaveHandler : IRequestHandler<SaveRequest, SaveResult>
{
    private readonly ISaveManager _saveManager;

    public SaveHandler(ISaveManager saveManager)
    {
        _saveManager = saveManager;
    }

    public SaveResult Invoke(SaveRequest request)
    {
        var success = _saveManager.Save(request.SlotName);
        return new SaveResult(success, success ? "저장 완료" : "저장 실패");
    }
}

// 사용
public class GamePresenter : IStartable
{
    private readonly IRequestHandler<SaveRequest, SaveResult> _saveHandler;

    void IStartable.Start()
    {
        var result = _saveHandler.Invoke(new SaveRequest("slot1"));
        Debug.Log(result.Message);
    }
}
```

### 비동기 핸들러

```csharp
public class AsyncSaveHandler : IAsyncRequestHandler<SaveRequest, SaveResult>
{
    public async ValueTask<SaveResult> InvokeAsync(
        SaveRequest request, CancellationToken ct = default)
    {
        var success = await SaveToCloudAsync(request.SlotName, ct);
        return new SaveResult(success, success ? "클라우드 저장 완료" : "저장 실패");
    }
}
```

### 등록

```csharp
// 동기 핸들러
builder.RegisterRequestHandler<SaveRequest, SaveResult, SaveHandler>(options);

// 비동기 핸들러
builder.RegisterAsyncRequestHandler<SaveRequest, SaveResult, AsyncSaveHandler>(options);
```

---

## 필터 (MessageHandlerFilter)

### 필터 구현

메시지 핸들링 전후에 공통 로직을 삽입한다.

```csharp
// 에러 무시 필터
public class IgnoreErrorFilter<T> : MessageHandlerFilter<T>
{
    public override void Handle(T message, Action<T> next)
    {
        try
        {
            next(message);
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"메시지 처리 에러 무시: {ex.Message}");
        }
    }
}

// 중복 값 필터
public class ChangedValueFilter<T> : MessageHandlerFilter<T>
{
    private T _lastValue;

    public override void Handle(T message, Action<T> next)
    {
        if (EqualityComparer<T>.Default.Equals(message, _lastValue))
            return;

        _lastValue = message;
        next(message);
    }
}
```

### 필터 등록

```csharp
// VContainer에 필터 등록
builder.RegisterMessageHandlerFilter<IgnoreErrorFilter<EnemyDefeated>>();

// 구독 시 개별 적용
subscriber.Subscribe(msg => Handle(msg),
    new ChangedValueFilter<int> { Order = 100 });
```

> Unity(IL2CPP)에서는 오픈 제네릭을 지원하지 않으므로,
> 글로벌 필터 등록 시 구체 타입으로 각각 등록해야 한다.

---

## Buffered Messages

마지막 발행 값을 보관하여 새 구독자가 즉시 최신 값을 수신한다.

```csharp
// 등록 (Buffered)
builder.RegisterMessageBroker<GameState>(options);  // 일반 등록으로도 Buffered 사용 가능

// 사용
public class StatusPresenter : IStartable, IDisposable
{
    private readonly IBufferedSubscriber<GameState> _stateSub;
    private IDisposable _disposable;

    void IStartable.Start()
    {
        var bag = DisposableBag.CreateBuilder();

        // 구독 즉시 마지막 발행 값을 수신한다
        _stateSub.Subscribe(state => UpdateUI(state)).AddTo(bag);

        _disposable = bag.Build();
    }

    void IDisposable.Dispose() => _disposable?.Dispose();
}
```

> Buffered는 키 기반(`IBufferedPublisher<TKey, TMessage>`)을 지원하지 않는다.
> 상태 관찰이 주 목적이면 R3 `ReactiveProperty`가 더 적합하다.

---

## EventFactory

DI 그룹과 독립적인 이벤트 인스턴스를 생성한다.
클래스 내부에서만 사용하는 이벤트에 적합하다.

```csharp
public class Timer : IDisposable
{
    private readonly IDisposablePublisher<float> _tickPublisher;
    public ISubscriber<float> OnTick { get; }

    public Timer(EventFactory eventFactory)
    {
        (_tickPublisher, OnTick) = eventFactory.CreateEvent<float>();
    }

    public void Update(float deltaTime)
    {
        _tickPublisher.Publish(deltaTime);
    }

    public void Dispose() => _tickPublisher.Dispose();
}
```

---

## 구독 관리

### DisposableBag 패턴 (표준)

```csharp
void IStartable.Start()
{
    var bag = DisposableBag.CreateBuilder();

    _subscriber1.Subscribe(msg => Handle1(msg)).AddTo(bag);
    _subscriber2.Subscribe(msg => Handle2(msg)).AddTo(bag);
    _subscriber3.Subscribe(msg => Handle3(msg)).AddTo(bag);

    _disposable = bag.Build();
}

void IDisposable.Dispose() => _disposable?.Dispose();
```

### 정적 생성 (구독 수가 고정일 때)

```csharp
var d1 = subscriber1.Subscribe(msg => Handle1(msg));
var d2 = subscriber2.Subscribe(msg => Handle2(msg));

_disposable = DisposableBag.Create(d1, d2);
```

### 조건부 구독 해제 (SingleAssignment)

핸들러 내부에서 자기 자신을 해제할 때 사용한다.

```csharp
var bag = DisposableBag.CreateBuilder();
var d = DisposableBag.CreateSingleAssignment();

subscriber.Subscribe(msg =>
{
    HandleOnce(msg);
    d.Dispose();  // 수신 후 즉시 구독 해제
}).SetTo(d).AddTo(bag);

_disposable = bag.Build();
```

---

## 디버깅

### MessagePipe Diagnostics 윈도우

Unity 에디터에서 `Window → MessagePipe Diagnostics`로 활성 구독을 시각적으로 확인한다.

**사용 조건:**
- `GlobalMessagePipe.SetProvider()` 호출 필요
- `options.EnableCaptureStackTrace = true`로 스택 트레이스 확인 가능 (개발 시에만)

```csharp
var options = builder.RegisterMessagePipe(opt =>
{
#if UNITY_EDITOR
    opt.EnableCaptureStackTrace = true;
#endif
});

builder.RegisterBuildCallback(c =>
    GlobalMessagePipe.SetProvider(c.AsServiceProvider()));
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `Subscribe()` 반환값 무시 | `DisposableBag`에 `.AddTo(bag)` |
| 상태 관찰에 MessagePipe 사용 | R3 `ReactiveProperty` 사용 |
| 메시지 타입을 클래스로 정의 | `readonly struct`로 정의 (할당 최소화) |
| 하나의 메시지 타입에 모든 정보 담기 | 목적별로 메시지 타입 분리 |
| `GlobalMessagePipe`를 DI 대신 남용 | DI 생성자 주입 기본, Global은 디버그용 |
| 동일 씬 내 1:1 통신에 MessagePipe 사용 | 직접 참조 또는 C# event |
| IL2CPP에서 오픈 제네릭 필터 등록 | 구체 타입으로 개별 등록 |
| `Publish` 호출 후 구독자 실행 순서에 의존 | 구독자 간 순서 보장 없음을 인지 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| Request/Response 패턴 상세, 다중 핸들러, 비동기 핸들러, 필터 체인 | `references/request-handler.md` |
| VContainer + MessagePipe + R3 통합, 시스템 간 통신 아키텍처 설계 | `references/architecture.md` |
