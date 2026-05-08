---
name: unity-unitask
description: >
  UniTask(비동기 처리)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 async/await 패턴, 취소 처리, PlayerLoop 타이밍을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - UniTask, UniTaskVoid, UniTask<T> 언급 시
  - async/await 기반 비동기 작업 (코루틴 대체)
  - CancellationToken, 타임아웃, 비동기 취소 관련 작업
  - UniTask.Delay, UniTask.WhenAll, UniTask.WhenAny 등 팩토리 메서드
  - PlayerLoopTiming, UniTask.Yield, UniTask.NextFrame 관련 작업
  - AsyncOperation, UnityWebRequest 비동기 대기
  - IUniTaskAsyncEnumerable, Channel 관련 작업

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - UniTask를 사용하지 않는 프로젝트
  - 동기 코드만으로 충분한 작업
  - R3 Observable 스트림 작업 (unity-r3 스킬 참조)
user-invocable: false
---

# Unity UniTask Skill

## 핵심 원칙

> **코루틴 대신 UniTask를 사용한다.**
> `IEnumerator` 코루틴은 사용하지 않는다. 모든 비동기 작업은 `async UniTask`로 작성한다.

> **async void는 절대 사용하지 않는다.**
> 반환값이 필요 없는 fire-and-forget에는 `async UniTaskVoid` 또는 `.Forget()`을 사용한다.

> **CancellationToken을 항상 전파한다.**
> 비동기 메서드는 `CancellationToken`을 매개변수로 받고, 내부 호출에 전달한다.

공식 저장소: https://github.com/Cysharp/UniTask

---

## UniTask vs 코루틴 vs Task

| | UniTask | 코루틴 (IEnumerator) | Task |
|---|---|---|---|
| 할당 | 제로 (struct) | GC 할당 | GC 할당 |
| 반환값 | `UniTask<T>` | 불가 | `Task<T>` |
| 예외 처리 | try/catch | 불가 | try/catch |
| 취소 | CancellationToken | 수동 플래그 | CancellationToken |
| 조합 | WhenAll/WhenAny | 불가 | WhenAll/WhenAny |
| PlayerLoop | 완전 지원 | 제한적 | 미지원 |
| WebGL | 지원 | 지원 | 미지원 (ThreadPool) |

---

## 기본 패턴

### async UniTask 메서드

```csharp
public async UniTask<int> LoadDataAsync(CancellationToken ct)
{
    var request = UnityWebRequest.Get("https://api.example.com/data");
    await request.SendWebRequest().WithCancellation(ct);

    if (request.result != UnityWebRequest.Result.Success)
        throw new Exception(request.error);

    return int.Parse(request.downloadHandler.text);
}
```

### Fire-and-Forget

```csharp
// 방법 1: async UniTaskVoid (이벤트 핸들러용)
async UniTaskVoid OnButtonClicked()
{
    await UniTask.Delay(TimeSpan.FromSeconds(1));
    Debug.Log("Clicked!");
}

// 방법 2: .Forget() (반환값을 무시할 때)
LoadDataAsync(destroyCancellationToken).Forget();

// 방법 3: 이벤트 등록
button.onClick.AddListener(UniTask.UnityAction(async () =>
{
    await UniTask.Delay(TimeSpan.FromSeconds(1));
}));
```

### MonoBehaviour에서의 사용

```csharp
public class MyComponent : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // destroyCancellationToken: GameObject 파괴 시 자동 취소
        await InitializeAsync(destroyCancellationToken);
    }

    private async UniTask InitializeAsync(CancellationToken ct)
    {
        var data = await LoadDataAsync(ct);
        await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: ct);
        ApplyData(data);
    }
}
```

---

## 취소 (CancellationToken)

### 기본 취소 패턴

```csharp
// MonoBehaviour 수명에 연결 (권장)
await DoSomethingAsync(destroyCancellationToken);

// 수동 취소
var cts = new CancellationTokenSource();
cancelButton.onClick.AddListener(() => cts.Cancel());

try
{
    await DoSomethingAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Debug.Log("취소됨");
}
```

### 취소 전파 (필수)

비동기 메서드는 CancellationToken을 받아서 내부 호출에 전달한다.

```csharp
public async UniTask ProcessAsync(CancellationToken ct)
{
    var data = await FetchAsync(ct);        // 전파
    await TransformAsync(data, ct);          // 전파
    await SaveAsync(data, ct);               // 전파
}
```

### OperationCanceledException 처리

```csharp
try
{
    await DoSomethingAsync(ct);
}
catch (OperationCanceledException)
{
    // 취소는 정상 흐름 — 에러 로깅 불필요
    return;
}
catch (Exception ex)
{
    // 실제 에러만 로깅
    Debug.LogException(ex);
}
```

### SuppressCancellationThrow (예외 없는 취소 확인)

성능이 중요한 곳에서 예외 발생을 억제한다.

```csharp
var (isCanceled, result) = await UniTask
    .Delay(TimeSpan.FromSeconds(3), cancellationToken: ct)
    .SuppressCancellationThrow();

if (isCanceled)
{
    // 취소 처리
    return;
}
// result 사용
```

---

## 타임아웃

### CancelAfterSlim (표준 CancelAfter 대신 사용)

```csharp
var cts = new CancellationTokenSource();
cts.CancelAfterSlim(TimeSpan.FromSeconds(5));  // PlayerLoop 기반, 스레드 미사용

try
{
    await request.SendWebRequest().WithCancellation(cts.Token);
}
catch (OperationCanceledException ex)
{
    if (ex.CancellationToken == cts.Token)
        Debug.Log("타임아웃");
}
```

### 링크된 토큰 (사용자 취소 + 타임아웃)

```csharp
public async UniTask<string> FetchWithTimeout(
    string url, CancellationToken userCt)
{
    using var timeoutCts = new CancellationTokenSource();
    timeoutCts.CancelAfterSlim(TimeSpan.FromSeconds(10));

    using var linkedCts = CancellationTokenSource
        .CreateLinkedTokenSource(userCt, timeoutCts.Token);

    try
    {
        var request = UnityWebRequest.Get(url);
        await request.SendWebRequest().WithCancellation(linkedCts.Token);
        return request.downloadHandler.text;
    }
    catch (OperationCanceledException)
    {
        if (timeoutCts.IsCancellationRequested)
            throw new TimeoutException($"요청 타임아웃: {url}");
        throw;  // 사용자 취소는 그대로 전파
    }
}
```

### TimeoutController (재사용)

반복되는 타임아웃에서 할당을 줄인다.

```csharp
private readonly TimeoutController _timeoutController = new();

public async UniTask<byte[]> DownloadAsync(string url, CancellationToken ct)
{
    try
    {
        var request = UnityWebRequest.Get(url);
        await request.SendWebRequest()
            .WithCancellation(_timeoutController.Timeout(TimeSpan.FromSeconds(5)));
        _timeoutController.Reset();  // 성공 시 리셋
        return request.downloadHandler.data;
    }
    catch (OperationCanceledException)
    {
        if (_timeoutController.IsTimeout())
            throw new TimeoutException();
        throw;
    }
}
```

---

## 팩토리 메서드

### 지연

```csharp
// 시간 기반 (TimeScale 적용)
await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: ct);

// 시간 기반 (TimeScale 무시)
await UniTask.Delay(TimeSpan.FromSeconds(1), ignoreTimeScale: true, cancellationToken: ct);

// 밀리초
await UniTask.Delay(500, cancellationToken: ct);

// 프레임 기반
await UniTask.DelayFrame(10, cancellationToken: ct);
```

### 프레임 동기화

```csharp
// 다음 프레임 (yield return null 대체)
await UniTask.NextFrame(ct);

// 현재 PlayerLoop 타이밍의 다음 호출
await UniTask.Yield(ct);

// 특정 타이밍
await UniTask.Yield(PlayerLoopTiming.PreLateUpdate, ct);

// FixedUpdate 대기
await UniTask.WaitForFixedUpdate(ct);

// 프레임 끝 대기 (Unity 2023.1+)
await UniTask.WaitForEndOfFrame(ct);
```

### 조건 대기

```csharp
// 조건이 true가 될 때까지
await UniTask.WaitUntil(() => isReady, cancellationToken: ct);

// 조건이 false가 될 때까지
await UniTask.WaitWhile(() => isLoading, cancellationToken: ct);

// 값이 변경될 때까지
await UniTask.WaitUntilValueChanged(transform, t => t.position, cancellationToken: ct);
```

### 작업 결합

```csharp
// 모두 완료 대기 (결과 튜플 반환)
var (data, config, user) = await UniTask.WhenAll(
    LoadDataAsync(ct),
    LoadConfigAsync(ct),
    LoadUserAsync(ct));

// 축약 문법
var (a, b, c) = await (taskA, taskB, taskC);

// 하나라도 완료되면 반환
var (winIndex, _) = await UniTask.WhenAny(
    WaitForInputAsync(ct),
    WaitForTimeoutAsync(ct));

// 완료되는 순서대로 처리 (Unity 9.0 스타일)
await foreach (var result in UniTask.WhenEach(task1, task2, task3))
{
    Debug.Log(result.GetResult());
}
```

---

## AsyncOperation 대기

### Unity 비동기 작업

```csharp
// 씬 로딩
await SceneManager.LoadSceneAsync("GameScene").WithCancellation(ct);

// 리소스 로딩
var asset = await Resources.LoadAsync<TextAsset>("data")
    .WithCancellation(ct);

// 웹 요청
var request = UnityWebRequest.Get(url);
await request.SendWebRequest().WithCancellation(ct);

// AssetBundle
var bundle = await AssetBundle.LoadFromFileAsync(path)
    .WithCancellation(ct);
```

### 진행률 보고

```csharp
var progress = Progress.Create<float>(ratio =>
    loadingBar.fillAmount = ratio);

await SceneManager.LoadSceneAsync("GameScene")
    .ToUniTask(progress: progress, cancellationToken: ct);
```

### Addressables

```csharp
var prefab = await Addressables.LoadAssetAsync<GameObject>("Enemy")
    .WithCancellation(ct);
```

### DOTween

`UNITASK_DOTWEEN_SUPPORT` 스크립팅 심볼 정의 필요.

```csharp
// 트윈 완료 대기
await transform.DOMoveX(10, 2f).WithCancellation(ct);

// 병렬 트윈
await UniTask.WhenAll(
    transform.DOMoveX(10, 2f).WithCancellation(ct),
    transform.DORotate(new Vector3(0, 180, 0), 2f).WithCancellation(ct));
```

---

## PlayerLoopTiming

UniTask는 Unity PlayerLoop의 특정 타이밍에서 실행된다.

### 주요 타이밍

| 타이밍 | 시점 | 용도 |
|---|---|---|
| `Initialization` | 프레임 시작 | 초기화 |
| `FixedUpdate` | 물리 업데이트 | 물리 연산 |
| `Update` | MonoBehaviour.Update 전 | 일반 로직 (기본값) |
| `PreLateUpdate` | LateUpdate 전 | 카메라 전 처리 |
| `PostLateUpdate` | 렌더링 후 | 스크린 캡처 |

### Yield vs NextFrame

```csharp
// Yield: 현재 타이밍의 "다음 호출"에서 재개
// → PreUpdate에서 호출하면 같은 프레임의 Update에서 재개될 수 있다
await UniTask.Yield(PlayerLoopTiming.Update);

// NextFrame: 항상 다음 프레임에서 재개 (yield return null과 동일)
await UniTask.NextFrame();
```

**기본은 `NextFrame`을 사용한다.** 같은 프레임 내 특정 타이밍으로 전환이 필요한 경우에만 `Yield`를 사용한다.

---

## UniTaskCompletionSource

콜백 기반 API를 async/await로 변환한다.

```csharp
public UniTask<bool> WaitForConfirmation(CancellationToken ct)
{
    var utcs = new UniTaskCompletionSource<bool>();

    confirmButton.onClick.AddListener(() => utcs.TrySetResult(true));
    cancelButton.onClick.AddListener(() => utcs.TrySetResult(false));

    ct.Register(() => utcs.TrySetCanceled());

    return utcs.Task;
}
```

---

## 스레딩

### ThreadPool 사용 (WebGL 미지원)

```csharp
// 무거운 계산을 백그라운드에서 실행
var result = await UniTask.RunOnThreadPool(() =>
{
    return HeavyCalculation();
});

// result는 메인 스레드에서 사용 가능
ApplyResult(result);
```

### 컨텍스트 전환

```csharp
await UniTask.SwitchToThreadPool();
// 여기는 ThreadPool
var data = ProcessData();

await UniTask.SwitchToMainThread();
// 여기는 메인 스레드
UpdateUI(data);
```

> **WebGL에서는 ThreadPool을 사용할 수 없다.**
> 크로스 플랫폼 코드에서는 `UniTask.RunOnThreadPool` 사용을 피한다.

---

## 에러 처리

### 전역 예외 핸들러

```csharp
UniTaskScheduler.UnobservedTaskException += (ex, _) =>
{
    Debug.LogException(ex);
};
```

### OperationCanceledException은 특별 취급

- `UnobservedTaskException`에서 자동으로 무시된다
- 취소는 에러가 아닌 정상 흐름이다
- `catch (OperationCanceledException)`으로 별도 처리한다

---

## 디버깅 (UniTaskTracker)

`Window → UniTask Tracker`에서 활성 UniTask를 시각적으로 확인한다.

- **Enable Tracking**: UniTask 추적 시작 (낮은 오버헤드)
- **Enable StackTrace**: 생성 위치 스택 트레이스 (높은 오버헤드, 디버깅 시에만)
- 프로덕션 빌드에서는 반드시 비활성화한다

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `async void` 사용 | `async UniTaskVoid` 또는 `.Forget()` |
| `CancellationToken` 전파 누락 | 모든 비동기 메서드에 `ct` 매개변수 전달 |
| `Task.Delay` 사용 | `UniTask.Delay` (PlayerLoop 기반, 제로 할당) |
| `CancelAfter()` 사용 | `CancelAfterSlim()` (스레드 미사용) |
| `new Progress<T>()` 사용 | `Progress.Create<T>()` (할당 최소화) |
| `UniTask`를 두 번 await | 한 번만 await 가능, 재사용 시 `UniTask.Lazy` |
| `await UniTask.Yield()` 다음 프레임 기대 | `await UniTask.NextFrame()` (확실한 다음 프레임) |
| 코루틴 `IEnumerator` 신규 작성 | `async UniTask`로 작성 |
| `Thread.Sleep()` 사용 | `await UniTask.Delay()` |
| `destroyCancellationToken` 무시 | MonoBehaviour에서는 항상 전달 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| AsyncEnumerable, Channel, 비동기 LINQ, 스트림 처리 | `references/async-streams.md` |
| 실전 비동기 패턴 (씬 전환, 리소스 로딩, 순차/병렬 실행) | `references/practical-patterns.md` |
