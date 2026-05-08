# AsyncEnumerable & Channel

## IUniTaskAsyncEnumerable

`using Cysharp.Threading.Tasks.Linq;` 필요.

시간에 따라 여러 값이 흐르는 비동기 스트림을 처리한다.

---

## 기본 사용법

### await foreach (C# 8.0+)

```csharp
await foreach (var _ in UniTaskAsyncEnumerable.EveryUpdate()
    .WithCancellation(ct))
{
    // 매 프레임 실행
    UpdatePosition();
}
```

### ForEachAsync (C# 7.3 호환)

```csharp
await UniTaskAsyncEnumerable.EveryUpdate()
    .ForEachAsync(_ => UpdatePosition(), ct);
```

### Subscribe (Fire-and-Forget)

```csharp
button.OnClickAsAsyncEnumerable()
    .Subscribe(_ => Debug.Log("Clicked"));
```

---

## 팩토리 메서드

### 프레임 기반

```csharp
UniTaskAsyncEnumerable.EveryUpdate()           // 매 Update
UniTaskAsyncEnumerable.EveryFixedUpdate()       // 매 FixedUpdate
UniTaskAsyncEnumerable.EveryLateUpdate()        // 매 LateUpdate
```

### 시간/프레임 카운트 기반

```csharp
UniTaskAsyncEnumerable.Timer(TimeSpan.FromSeconds(3))         // 3초 후 단일 발행
UniTaskAsyncEnumerable.Interval(TimeSpan.FromSeconds(0.5))    // 0.5초마다 발행
UniTaskAsyncEnumerable.TimerFrame(60)                          // 60프레임 후 단일 발행
UniTaskAsyncEnumerable.IntervalFrame(30)                       // 30프레임마다 발행
```

### 값 변경 감지

```csharp
UniTaskAsyncEnumerable.EveryValueChanged(transform, t => t.position)
```

### 커스텀 생성

```csharp
UniTaskAsyncEnumerable.Create<int>(async (writer, ct) =>
{
    for (int i = 0; i < 10; i++)
    {
        await writer.YieldAsync(i);
        await UniTask.Delay(TimeSpan.FromSeconds(0.5), cancellationToken: ct);
    }
});
```

---

## LINQ 오퍼레이터

### 필터링

```csharp
source.Where(x => x > 0)
source.OfType<Derived>()
source.Distinct()
source.DistinctUntilChanged()
source.Take(5)
source.TakeUntil(ct)
source.Skip(2)
source.SkipUntil(ct)
```

### 변환

```csharp
source.Select(x => x * 2)
source.SelectMany(x => GetItemsAsync(x))
source.SelectAwait(async x => await FetchAsync(x))
source.SelectAwaitWithCancellation(async (x, ct) => await FetchAsync(x, ct))
source.Cast<Derived>()
```

### 집계

```csharp
await source.CountAsync(ct)
await source.SumAsync(ct)
await source.MinAsync(ct)
await source.MaxAsync(ct)
await source.AverageAsync(ct)
await source.FirstAsync(ct)
await source.LastAsync(ct)
await source.ToArrayAsync(ct)
await source.ToListAsync(ct)
```

### 결합

```csharp
source.CombineLatest(other, (a, b) => (a, b))
source.Merge(other)
source.Zip(other, (a, b) => (a, b))
```

### 버퍼링

```csharp
source.Buffer(count: 10)                        // 10개씩 묶기
source.Buffer(TimeSpan.FromSeconds(1))           // 1초 구간 묶기
```

### 유틸리티

```csharp
source.Do(x => Debug.Log(x))                    // 사이드 이펙트
source.Queue()                                    // 비동기 처리 중 이벤트 큐잉
source.Publish()                                  // 멀티캐스트
```

---

## uGUI 이벤트 스트림

```csharp
// Button
button.OnClickAsAsyncEnumerable()

// InputField
inputField.OnValueChangedAsAsyncEnumerable()
inputField.OnEndEditAsAsyncEnumerable()

// Slider
slider.OnValueChangedAsAsyncEnumerable()

// Toggle
toggle.OnValueChangedAsAsyncEnumerable()

// Dropdown
dropdown.OnValueChangedAsAsyncEnumerable()
```

### 예시: 검색 입력

```csharp
await inputField.OnValueChangedAsAsyncEnumerable()
    .Where(text => text.Length >= 2)
    .Debounce(TimeSpan.FromMilliseconds(300))
    .SelectAwaitWithCancellation(async (text, ct) =>
        await SearchAsync(text, ct))
    .ForEachAsync(results => ShowResults(results), ct);
```

---

## MonoBehaviour 트리거

`using Cysharp.Threading.Tasks.Triggers;` 필요.

```csharp
// 물리 이벤트
this.GetAsyncOnCollisionEnterTrigger()
    .ForEachAsync(col => HandleCollision(col), ct).Forget();

this.GetAsyncOnTriggerEnterTrigger()
    .ForEachAsync(col => HandleTrigger(col), ct).Forget();

// 수명 이벤트
this.GetAsyncOnEnableTrigger()
this.GetAsyncOnDisableTrigger()
this.GetAsyncOnDestroyTrigger()

// 프레임 이벤트
this.GetAsyncUpdateTrigger()
this.GetAsyncFixedUpdateTrigger()
this.GetAsyncLateUpdateTrigger()
```

---

## Channel

비동기 Producer-Consumer 패턴. `System.Threading.Channels`의 UniTask 버전.

```csharp
// 채널 생성 (단일 소비자, 무제한 버퍼)
var channel = Channel.CreateSingleConsumerUnbounded<int>();

// Producer
async UniTask ProduceAsync(ChannelWriter<int> writer, CancellationToken ct)
{
    for (int i = 0; i < 100; i++)
    {
        await UniTask.Delay(TimeSpan.FromMilliseconds(100), cancellationToken: ct);
        writer.TryWrite(i);
    }
    writer.TryComplete();
}

// Consumer
async UniTask ConsumeAsync(ChannelReader<int> reader, CancellationToken ct)
{
    await foreach (var item in reader.ReadAllAsync().WithCancellation(ct))
    {
        Debug.Log($"Received: {item}");
    }
}

// 실행
ProduceAsync(channel.Writer, ct).Forget();
await ConsumeAsync(channel.Reader, ct);
```

### Channel vs MessagePipe

| | Channel | MessagePipe |
|---|---|---|
| 패턴 | Producer-Consumer (1:1) | Pub/Sub (1:N) |
| 버퍼 | 내부 큐 보유 | 즉시 전달 |
| 백프레셔 | 지원 (버퍼 제한 시) | 미지원 |
| 용도 | 작업 큐, 파이프라인 | 이벤트 브로드캐스트 |

---

## AsyncReactiveProperty

비동기 대기가 가능한 ReactiveProperty. R3 없이 간단한 상태 관찰이 필요할 때 사용한다.

```csharp
var hp = new AsyncReactiveProperty<int>(100);

// 값 변경 구독
hp.ForEachAsync(value => Debug.Log($"HP: {value}"), ct).Forget();

// 값 설정 (구독자에게 통지)
hp.Value = 80;

// 다음 변경 대기
var nextValue = await hp.WaitAsync(ct);

// 초기값 제외
await hp.WithoutCurrent()
    .ForEachAsync(value => Debug.Log($"Changed: {value}"), ct);
```

> R3를 사용하는 프로젝트에서는 R3의 `ReactiveProperty<T>`를 사용한다.
> `AsyncReactiveProperty`는 R3 없이 UniTask만 사용하는 프로젝트에서 사용한다.
