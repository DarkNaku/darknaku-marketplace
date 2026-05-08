# 실전 비동기 패턴

## 씬 전환

### 페이드 + 씬 로딩

```csharp
public class SceneTransitionService
{
    private readonly FadeView _fadeView;

    public async UniTask TransitionAsync(string sceneName, CancellationToken ct)
    {
        // 페이드 아웃
        await _fadeView.FadeOutAsync(ct);

        // 씬 로딩 (진행률 보고)
        var progress = Progress.Create<float>(ratio =>
            _fadeView.SetProgress(ratio));

        await SceneManager.LoadSceneAsync(sceneName)
            .ToUniTask(progress: progress, cancellationToken: ct);

        // 페이드 인
        await _fadeView.FadeInAsync(ct);
    }
}
```

### 여러 씬 Additive 로딩

```csharp
public async UniTask LoadGameScenesAsync(CancellationToken ct)
{
    // 병렬 로딩
    await UniTask.WhenAll(
        SceneManager.LoadSceneAsync("Environment", LoadSceneMode.Additive)
            .WithCancellation(ct),
        SceneManager.LoadSceneAsync("UI", LoadSceneMode.Additive)
            .WithCancellation(ct),
        SceneManager.LoadSceneAsync("Audio", LoadSceneMode.Additive)
            .WithCancellation(ct));
}
```

---

## 리소스 로딩

### 순차 로딩 (의존성 있을 때)

```csharp
public async UniTask<GameData> LoadGameDataAsync(CancellationToken ct)
{
    var config = await Addressables.LoadAssetAsync<GameConfig>("config")
        .WithCancellation(ct);

    var levelData = await Addressables.LoadAssetAsync<LevelData>(
        config.FirstLevelKey).WithCancellation(ct);

    return new GameData(config, levelData);
}
```

### 병렬 로딩 (독립적일 때)

```csharp
public async UniTask<(Sprite[], AudioClip[])> LoadAssetsAsync(
    CancellationToken ct)
{
    var (sprites, clips) = await UniTask.WhenAll(
        LoadSpritesAsync(ct),
        LoadAudioClipsAsync(ct));

    return (sprites, clips);
}
```

### 순차 + 병렬 조합

```csharp
public async UniTask InitializeAsync(CancellationToken ct)
{
    // 1단계: 설정 로드 (순차)
    var config = await LoadConfigAsync(ct);

    // 2단계: 설정에 따른 에셋 병렬 로드
    var (sprites, sounds, prefabs) = await UniTask.WhenAll(
        LoadSpritesAsync(config.SpriteKeys, ct),
        LoadSoundsAsync(config.SoundKeys, ct),
        LoadPrefabsAsync(config.PrefabKeys, ct));

    // 3단계: 초기화
    Initialize(sprites, sounds, prefabs);
}
```

---

## 재시도 패턴

### 지수 백오프 재시도

```csharp
public async UniTask<T> RetryAsync<T>(
    Func<CancellationToken, UniTask<T>> operation,
    int maxRetries,
    CancellationToken ct)
{
    for (int i = 0; i <= maxRetries; i++)
    {
        try
        {
            return await operation(ct);
        }
        catch (Exception ex) when (i < maxRetries
            && ex is not OperationCanceledException)
        {
            var delay = TimeSpan.FromSeconds(Math.Pow(2, i));
            Debug.LogWarning($"재시도 {i + 1}/{maxRetries}: {delay.TotalSeconds}초 후");
            await UniTask.Delay(delay, cancellationToken: ct);
        }
    }

    throw new InvalidOperationException("도달 불가");
}

// 사용
var data = await RetryAsync(ct => FetchDataAsync(url, ct), maxRetries: 3, ct);
```

---

## 대기 패턴

### 여러 조건 중 하나 대기

```csharp
public async UniTask<InputResult> WaitForInputAsync(CancellationToken ct)
{
    var (winIndex, _) = await UniTask.WhenAny(
        WaitForKeyPressAsync(KeyCode.Space, ct),
        WaitForButtonClickAsync(confirmButton, ct),
        WaitForTimeoutAsync(TimeSpan.FromSeconds(30), ct));

    return winIndex switch
    {
        0 => InputResult.KeyPress,
        1 => InputResult.ButtonClick,
        2 => InputResult.Timeout,
        _ => throw new InvalidOperationException()
    };
}

private async UniTask WaitForKeyPressAsync(KeyCode key, CancellationToken ct)
{
    await UniTask.WaitUntil(() => Input.GetKeyDown(key), cancellationToken: ct);
}

private async UniTask WaitForButtonClickAsync(Button button, CancellationToken ct)
{
    await button.OnClickAsync(ct);
}

private async UniTask WaitForTimeoutAsync(TimeSpan duration, CancellationToken ct)
{
    await UniTask.Delay(duration, cancellationToken: ct);
}
```

### 순차 다이얼로그

```csharp
public async UniTask ShowDialogSequenceAsync(
    DialogData[] dialogs, CancellationToken ct)
{
    foreach (var dialog in dialogs)
    {
        _dialogView.ShowText(dialog.Text);

        // 다음 버튼 또는 자동 진행 대기
        await UniTask.WhenAny(
            _dialogView.WaitForNextAsync(ct),
            UniTask.Delay(dialog.AutoAdvanceTime, cancellationToken: ct));
    }
}
```

---

## VContainer EntryPoint에서의 사용

### IAsyncStartable

```csharp
public class GameInitializer : IAsyncStartable, IDisposable
{
    private readonly IResourceLoader _loader;
    private readonly GameView _view;
    private CancellationTokenSource _cts;

    public GameInitializer(IResourceLoader loader, GameView view)
    {
        _loader = loader;
        _view = view;
    }

    public async UniTask StartAsync(CancellationToken ct)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(ct);

        _view.ShowLoading();

        var (config, assets) = await UniTask.WhenAll(
            _loader.LoadConfigAsync(_cts.Token),
            _loader.LoadAssetsAsync(_cts.Token));

        _view.HideLoading();
        _view.Initialize(config, assets);
    }

    public void Dispose() => _cts?.Cancel();
}
```

등록:

```csharp
builder.RegisterEntryPoint<GameInitializer>();
```

---

## 폴링 패턴

### 주기적 상태 확인

```csharp
public async UniTask PollServerStatusAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        try
        {
            var status = await FetchStatusAsync(ct);
            UpdateStatusUI(status);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            Debug.LogWarning($"상태 확인 실패: {ex.Message}");
        }

        await UniTask.Delay(TimeSpan.FromSeconds(30), cancellationToken: ct);
    }
}
```

---

## 동시성 제한

### 세마포어를 활용한 동시 요청 제한

```csharp
private readonly SemaphoreSlim _semaphore = new(maxConcurrency: 3);

public async UniTask<T> ThrottledRequestAsync<T>(
    Func<CancellationToken, UniTask<T>> request, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        return await request(ct);
    }
    finally
    {
        _semaphore.Release();
    }
}

// 사용: 최대 3개 동시 다운로드
var tasks = urls.Select(url =>
    ThrottledRequestAsync(ct => DownloadAsync(url, ct), ct));
var results = await UniTask.WhenAll(tasks);
```
