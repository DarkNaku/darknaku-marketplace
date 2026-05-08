---
name: unity-r3
description: >
  R3(Reactive Extensions)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 Observable 패턴, 구독 관리, Unity 연동 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - R3, Observable, Observer, Subject, ReactiveProperty 언급 시
  - Subscribe, OnNext, OnCompleted, Disposable 관련 작업
  - 이벤트 스트림, 리액티브 바인딩, 데이터 바인딩 관련 작업
  - EveryUpdate, EveryValueChanged 등 프레임 기반 Observable 사용 시
  - UniRx에서 R3로 마이그레이션 작업

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - R3를 사용하지 않는 프로젝트
  - 단순 C# event나 Action/Func 콜백만 사용하는 경우
  - async/await만으로 충분한 비동기 작업
user-invocable: false
---

# Unity R3 Skill

## 핵심 원칙

> **R3는 이벤트 스트림(LINQ to Events)을 위한 라이브러리다.**
> 비동기 작업 하나를 기다리는 용도에는 async/await를 사용한다.
> Observable은 시간에 따라 여러 값이 흐르는 이벤트 스트림에 사용한다.

> **구독은 반드시 해제한다.**
> 모든 `Subscribe` 호출에는 대응하는 Dispose 전략이 있어야 한다.
> `AddTo(this)` 또는 `AddTo(destroyCancellationToken)`으로 MonoBehaviour 수명에 연결한다.

공식 저장소: https://github.com/Cysharp/R3

---

## R3 vs UniRx 핵심 차이

R3는 UniRx의 후속이 아닌 재설계다. 기존 UniRx 사용자는 다음 차이를 반드시 인지한다.

| UniRx | R3 | 변경 이유 |
|---|---|---|
| `OnError`로 스트림 중단 | `OnErrorResume`으로 계속 진행 | 파이프라인 중단은 대부분 의도치 않은 동작 |
| `IScheduler` | `TimeProvider` / `FrameProvider` | .NET 표준 통합, 테스트 용이 |
| `ReactiveProperty<T>.Value` set 시 항상 발행 | 중복 값 무시 (DistinctUntilChanged 내장) | 불필요한 업데이트 방지 |
| `CompositeDisposable` | `DisposableBag` (구조체) | 할당 최소화 |
| `AddTo(gameObject)` | `AddTo(this)` 또는 `AddTo(destroyCancellationToken)` | Component 기반 정리 |
| `Observable.EveryUpdate()` 전역 | `Observable.EveryUpdate(cancellationToken)` | 명시적 수명 관리 |

---

## 구독 관리

### AddTo 패턴 (필수)

모든 구독은 수명 관리 대상에 연결한다.

```csharp
// MonoBehaviour에 연결 (권장)
Observable.EveryUpdate()
    .Subscribe(_ => DoSomething())
    .AddTo(this);

// destroyCancellationToken에 연결
Observable.EveryUpdate(destroyCancellationToken)
    .Subscribe(_ => DoSomething());

// GameObject에 연결
observable.Subscribe(x => Handle(x))
    .AddTo(gameObject);
```

### DisposableBag (순수 C# 클래스에서)

MonoBehaviour가 아닌 순수 C# 클래스에서는 `DisposableBag`을 사용한다.

```csharp
public class GamePresenter : IDisposable
{
    private DisposableBag _disposables;

    public GamePresenter(IGameModel model, GameView view)
    {
        model.Score
            .Subscribe(score => view.SetScore(score))
            .AddTo(ref _disposables);

        model.Health
            .Subscribe(hp => view.SetHealth(hp))
            .AddTo(ref _disposables);
    }

    public void Dispose() => _disposables.Dispose();
}
```

### Disposable.CreateBuilder (여러 구독 묶기)

구독 수가 많을 때 빌더 패턴으로 효율적으로 묶는다.

```csharp
var d = Disposable.CreateBuilder();
observable1.Subscribe(x => Handle1(x)).AddTo(ref d);
observable2.Subscribe(x => Handle2(x)).AddTo(ref d);
observable3.Subscribe(x => Handle3(x)).AddTo(ref d);
_disposable = d.Build();
```

---

## ReactiveProperty

### 기본 사용법

값의 변경을 Observable로 관찰할 수 있는 프로퍼티. **중복 값은 자동으로 무시된다.**

```csharp
public class PlayerModel
{
    public ReactiveProperty<int> Hp { get; } = new(100);
    public ReactiveProperty<string> Name { get; } = new("Player");

    public void TakeDamage(int damage)
    {
        Hp.Value -= damage;  // 값이 변경될 때만 구독자에게 통지
    }
}
```

### ReadOnlyReactiveProperty (외부 노출용)

외부에서 값을 변경하지 못하도록 읽기 전용으로 노출한다.

```csharp
public class PlayerModel
{
    private readonly ReactiveProperty<int> _hp = new(100);
    public ReadOnlyReactiveProperty<int> Hp => _hp;

    // 파생 프로퍼티
    public ReadOnlyReactiveProperty<bool> IsDead { get; }

    public PlayerModel()
    {
        IsDead = _hp.Select(hp => hp <= 0).ToReadOnlyReactiveProperty();
    }
}
```

### SerializableReactiveProperty (Inspector 연동)

Inspector에서 편집 가능한 ReactiveProperty. MonoBehaviour의 필드로 사용한다.

```csharp
public class EnemyConfig : MonoBehaviour
{
    public SerializableReactiveProperty<int> maxHp = new(100);
    public SerializableReactiveProperty<float> speed = new(5f);
}
```

---

## Subject

### Subject<T> (기본 이벤트 발행)

수동으로 값을 발행하는 Observable이자 Observer.

```csharp
public class GameEvents
{
    private readonly Subject<int> _onScoreChanged = new();
    public Observable<int> OnScoreChanged => _onScoreChanged;

    public void AddScore(int points)
    {
        _onScoreChanged.OnNext(points);
    }

    // Subject는 Dispose 시 OnCompleted를 자동 발행한다
}
```

> **Subject.OnNext는 스레드 세이프하지 않다.**
> 멀티스레드에서 호출해야 하면 lock을 사용하거나 `Synchronize()`를 적용한다.

### ReactiveProperty vs Subject

| | ReactiveProperty | Subject |
|---|---|---|
| 현재 값 보유 | O (`.Value`) | X |
| 중복 값 무시 | O | X |
| 초기값 발행 | O (구독 시 현재 값 발행) | X |
| 용도 | 상태 표현 (HP, 점수 등) | 이벤트 발행 (클릭, 충돌 등) |

---

## 프레임 기반 Observable

### EveryUpdate

매 프레임 실행되는 Observable. 반드시 수명을 지정한다.

```csharp
// CancellationToken으로 수명 관리
Observable.EveryUpdate(destroyCancellationToken)
    .Subscribe(_ => UpdatePosition());

// AddTo로 수명 관리
Observable.EveryUpdate()
    .Subscribe(_ => UpdatePosition())
    .AddTo(this);
```

### EveryValueChanged

프로퍼티 값이 변경될 때만 발행한다. 매 프레임 폴링하되 변경 시에만 통지한다.

```csharp
// Transform 위치 변경 감지
Observable.EveryValueChanged(this, x => x.transform.position)
    .Subscribe(pos => UpdateUI(pos))
    .AddTo(this);
```

### 프레임 기반 타이밍 오퍼레이터

시간 기반 오퍼레이터에는 프레임 기반 변형이 있다.

| 시간 기반 | 프레임 기반 | 용도 |
|---|---|---|
| `Delay(TimeSpan)` | `DelayFrame(int)` | 지연 |
| `Debounce(TimeSpan)` | `DebounceFrame(int)` | 연속 입력 억제 |
| `ThrottleFirst(TimeSpan)` | `ThrottleFirstFrame(int)` | 첫 입력만 허용 |
| `ThrottleLast(TimeSpan)` | `ThrottleLastFrame(int)` | 마지막 입력만 허용 |
| `Timer(TimeSpan)` | `TimerFrame(int)` | 단일 타이머 |
| `Interval(TimeSpan)` | `IntervalFrame(int)` | 주기적 발행 |
| `Sample(TimeSpan)` | `SampleFrame(int)` | 주기적 샘플링 |

---

## Unity UI 바인딩

### uGUI 바인딩

```csharp
// Button 클릭
button.OnClickAsObservable()
    .Subscribe(_ => HandleClick())
    .AddTo(this);

// Toggle 값 변경
toggle.OnValueChangedAsObservable()
    .Subscribe(isOn => HandleToggle(isOn))
    .AddTo(this);

// InputField 텍스트 변경
inputField.OnValueChangedAsObservable()
    .Subscribe(text => HandleInput(text))
    .AddTo(this);

// Observable → Text 바인딩
model.Score
    .Select(s => $"Score: {s}")
    .SubscribeToText(scoreText)
    .AddTo(this);
```

### MonoBehaviour 트리거

MonoBehaviour 이벤트를 Observable로 변환한다.

```csharp
// 충돌 감지
this.OnCollisionEnterAsObservable()
    .Subscribe(collision => HandleCollision(collision))
    .AddTo(this);

// 트리거 감지
this.OnTriggerEnterAsObservable()
    .Subscribe(other => HandleTrigger(other))
    .AddTo(this);
```

---

## 주요 오퍼레이터 활용

### 필터링

```csharp
// 조건 필터
source.Where(x => x > 0)

// 중복 제거 (ReactiveProperty는 내장)
source.DistinctUntilChanged()

// 처음 N개만
source.Take(5)

// 조건 충족까지
source.TakeUntil(onGameOver)

// N개 건너뛰기
source.Skip(1)
```

### 변환

```csharp
// 값 변환
source.Select(x => x * 2)

// 비동기 변환
source.SelectAwait(async (x, ct) => await FetchData(x, ct), AwaitOperation.Drop)

// 평탄화
source.SelectMany(x => GetItems(x))
```

### 결합

```csharp
// 최신 값 결합 (여러 입력 → 하나의 출력)
Observable.CombineLatest(hp, maxHp, (current, max) => (float)current / max)
    .Subscribe(ratio => healthBar.SetRatio(ratio))
    .AddTo(this);

// 다른 Observable의 최신 값과 결합
source.WithLatestFrom(config, (value, cfg) => Process(value, cfg))

// 여러 스트림 병합
Observable.Merge(source1, source2, source3)
    .Subscribe(x => Handle(x))
    .AddTo(this);
```

### 타이밍

```csharp
// 연속 입력 억제 (검색 입력 등)
inputField.OnValueChangedAsObservable()
    .Debounce(TimeSpan.FromMilliseconds(300))
    .SubscribeAwait(async (text, ct) => await Search(text, ct), AwaitOperation.Switch)
    .AddTo(this);

// 연타 방지
button.OnClickAsObservable()
    .ThrottleFirst(TimeSpan.FromSeconds(1))
    .Subscribe(_ => HandleClick())
    .AddTo(this);
```

### 비동기 연동 (AwaitOperation)

비동기 작업과 Observable을 결합할 때 동시성 제어 방식을 지정한다.

| AwaitOperation | 동작 | 용도 |
|---|---|---|
| `Sequential` | 큐잉하여 순서대로 실행 | 순서가 중요한 처리 |
| `Drop` | 실행 중이면 새 값 무시 | 버튼 연타 방지 |
| `Switch` | 이전 작업 취소, 새 작업 시작 | 검색 자동완성 |
| `Parallel` | 모든 값 동시 실행 | 독립적인 비동기 처리 |

```csharp
// 검색: 이전 요청 취소하고 새 요청 실행
searchInput.OnValueChangedAsObservable()
    .Debounce(TimeSpan.FromMilliseconds(300))
    .SelectAwait(async (query, ct) => await SearchApi(query, ct), AwaitOperation.Switch)
    .Subscribe(results => UpdateSearchResults(results))
    .AddTo(this);
```

---

## 에러 처리

### OnErrorResume (R3의 기본 동작)

R3에서 에러는 스트림을 중단하지 않는다. `OnErrorResume`으로 전달되어 처리 후 계속 진행한다.

```csharp
source
    .Subscribe(
        onNext: x => Handle(x),
        onErrorResume: ex => Debug.LogWarning($"에러 발생, 계속 진행: {ex}"),
        onCompleted: result =>
        {
            if (result.IsFailure)
                Debug.LogError($"스트림 실패 종료: {result.Exception}");
        })
    .AddTo(this);
```

### 전역 예외 핸들러

처리되지 않은 예외의 기본 핸들러를 등록한다.

```csharp
ObservableSystem.RegisterUnhandledExceptionHandler(ex =>
{
    Debug.LogException(ex);
});
```

---

## TimeProvider / FrameProvider

### Unity 전용 Provider

| Provider | 용도 |
|---|---|
| `UnityTimeProvider.Update` | Update 루프 기준 시간 |
| `UnityTimeProvider.UpdateIgnoreTimeScale` | TimeScale 무시 |
| `UnityTimeProvider.FixedUpdate` | FixedUpdate 루프 기준 시간 |
| `UnityTimeProvider.Realtime` | 실시간 (TimeScale 무시) |
| `UnityFrameProvider.Update` | Update 프레임 카운트 |
| `UnityFrameProvider.FixedUpdate` | FixedUpdate 프레임 카운트 |

### 테스트에서 사용

```csharp
var fakeTime = new FakeTimeProvider();
var list = Observable.Timer(TimeSpan.FromSeconds(5), fakeTime).ToLiveList();

fakeTime.Advance(TimeSpan.FromSeconds(4));
list.AssertIsNotCompleted();

fakeTime.Advance(TimeSpan.FromSeconds(1));
list.AssertIsCompleted();
```

```csharp
var fakeFrame = new FakeFrameProvider();
var list = Observable.EveryUpdate(fakeFrame)
    .Select(_ => fakeFrame.GetFrameCount())
    .ToLiveList();

fakeFrame.Advance(3);
list.AssertEqual([0, 1, 2]);
```

---

## 구독 추적 (디버깅)

### ObservableTracker

활성 구독을 추적하여 구독 누수를 탐지한다.

```csharp
ObservableTracker.EnableTracking = true;
ObservableTracker.EnableStackTrace = true;  // 스택 트레이스 포함 (성능 비용 있음)
```

Unity 에디터에서 `Window → Observable Tracker`로 시각적으로 확인할 수 있다.

---

## 스레드 안전성

- `Subject<T>.OnNext`는 **스레드 세이프하지 않다** — 멀티스레드 접근 시 lock 사용
- `ReactiveProperty<T>`는 **스레드 세이프하지 않다** — 멀티스레드 필요 시 `SynchronizedReactiveProperty<T>` 사용
- `BehaviorSubject<T>`, `ReplaySubject<T>`는 **스레드 세이프하다**
- 오퍼레이터 체인은 스레드 세이프하지만, OnNext 호출은 단일 스레드를 가정한다

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `Subscribe()` 후 Dispose 전략 없음 | `AddTo(this)` 또는 `AddTo(destroyCancellationToken)` |
| `EveryUpdate()` 수명 미지정 | `EveryUpdate(destroyCancellationToken)` 또는 `.AddTo(this)` |
| 단일 비동기 작업에 Observable 사용 | `async/await` 사용 |
| `Subject`를 외부에 직접 노출 | `Observable<T>`로 캐스팅하여 노출 |
| `ReactiveProperty`를 이벤트 발행용으로 사용 | 이벤트는 `Subject`, 상태는 `ReactiveProperty` |
| `CompositeDisposable` (할당 비용) | `DisposableBag` (구조체, 저할당) |
| `OnError`로 스트림 중단 기대 (UniRx 습관) | `OnErrorResume` + `OnCompleted(Result)` 패턴 |
| `ReactiveProperty.Value` 설정으로 강제 발행 기대 | 중복 값은 무시됨, 강제 발행은 `.OnNext(value)` |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| 팩토리 메서드, 전체 오퍼레이터 목록, 오퍼레이터 선택 기준 | `references/operators.md` |
| VContainer + R3 + MVP 통합 패턴, Presenter에서의 구독 관리 | `references/di-integration.md` |
