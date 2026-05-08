# Addressables 로딩 실전 패턴

## 프리로딩 (사전 로딩)

### 게임 시작 시 필수 에셋 프리로드

```csharp
public class AssetPreloader : IAsyncStartable, IDisposable
{
    private readonly List<AsyncOperationHandle> _handles = new();

    public async UniTask StartAsync(CancellationToken ct)
    {
        // 라벨로 일괄 프리로드
        var uiHandle = Addressables.LoadAssetsAsync<GameObject>(
            "preload-ui", null);
        _handles.Add(uiHandle);

        var audioHandle = Addressables.LoadAssetsAsync<AudioClip>(
            "preload-audio", null);
        _handles.Add(audioHandle);

        var configHandle = Addressables.LoadAssetsAsync<ScriptableObject>(
            "preload-config", null);
        _handles.Add(configHandle);

        // 병렬 로딩
        await UniTask.WhenAll(
            uiHandle.WithCancellation(ct),
            audioHandle.WithCancellation(ct),
            configHandle.WithCancellation(ct));
    }

    public void Dispose()
    {
        foreach (var handle in _handles)
        {
            if (handle.IsValid())
                Addressables.Release(handle);
        }
        _handles.Clear();
    }
}
```

### 진행률 보고 프리로드

```csharp
public async UniTask PreloadWithProgressAsync(
    string[] labels, IProgress<float> progress, CancellationToken ct)
{
    var handles = new List<AsyncOperationHandle>();

    foreach (var label in labels)
    {
        handles.Add(Addressables.LoadAssetsAsync<Object>(label, null));
    }

    while (!handles.All(h => h.IsDone))
    {
        float totalProgress = handles.Sum(h => h.PercentComplete) / handles.Count;
        progress.Report(totalProgress);
        await UniTask.Yield(ct);
    }

    progress.Report(1f);
}
```

---

## 오브젝트 풀 연동

### Addressable 프리팹 풀

```csharp
public class AddressablePool : IDisposable
{
    private readonly string _address;
    private readonly Queue<GameObject> _pool = new();
    private readonly List<GameObject> _active = new();
    private AsyncOperationHandle<GameObject> _prefabHandle;
    private bool _isLoaded;

    public AddressablePool(string address)
    {
        _address = address;
    }

    public async UniTask WarmupAsync(int count, CancellationToken ct)
    {
        _prefabHandle = Addressables.LoadAssetAsync<GameObject>(_address);
        await _prefabHandle.WithCancellation(ct);
        _isLoaded = true;

        for (int i = 0; i < count; i++)
        {
            var obj = Object.Instantiate(_prefabHandle.Result);
            obj.SetActive(false);
            _pool.Enqueue(obj);
        }
    }

    public GameObject Get(Vector3 position, Quaternion rotation)
    {
        GameObject obj;

        if (_pool.Count > 0)
        {
            obj = _pool.Dequeue();
        }
        else
        {
            obj = Object.Instantiate(_prefabHandle.Result);
        }

        obj.transform.SetPositionAndRotation(position, rotation);
        obj.SetActive(true);
        _active.Add(obj);
        return obj;
    }

    public void Return(GameObject obj)
    {
        obj.SetActive(false);
        _active.Remove(obj);
        _pool.Enqueue(obj);
    }

    public void Dispose()
    {
        foreach (var obj in _pool)
            Object.Destroy(obj);
        foreach (var obj in _active)
            Object.Destroy(obj);

        _pool.Clear();
        _active.Clear();

        if (_prefabHandle.IsValid())
            Addressables.Release(_prefabHandle);
    }
}
```

---

## 씬 전환 패턴

### 페이드 + Addressable 씬 전환

```csharp
public class SceneTransitionService : IDisposable
{
    private AsyncOperationHandle<SceneInstance> _currentSceneHandle;

    public async UniTask TransitionToAsync(
        string sceneAddress, FadeView fadeView, CancellationToken ct)
    {
        // 1. 페이드 아웃
        await fadeView.FadeOutAsync(ct);

        // 2. 이전 씬 언로드
        if (_currentSceneHandle.IsValid())
        {
            await Addressables.UnloadSceneAsync(_currentSceneHandle)
                .WithCancellation(ct);
        }

        // 3. 새 씬 로드 (활성화 지연)
        _currentSceneHandle = Addressables.LoadSceneAsync(
            sceneAddress, LoadSceneMode.Additive, activateOnLoad: false);

        // 4. 로딩 진행률 표시
        while (!_currentSceneHandle.IsDone)
        {
            fadeView.SetProgress(_currentSceneHandle.PercentComplete);
            await UniTask.Yield(ct);
        }

        // 5. 씬 활성화
        await _currentSceneHandle.Result.ActivateAsync();

        // 6. 페이드 인
        await fadeView.FadeInAsync(ct);
    }

    public void Dispose()
    {
        if (_currentSceneHandle.IsValid())
            Addressables.UnloadSceneAsync(_currentSceneHandle);
    }
}
```

### VContainer + Additive 씬

```csharp
public class GameLifetimeScope : LifetimeScope
{
    private AsyncOperationHandle<SceneInstance> _uiSceneHandle;

    protected override void Configure(IContainerBuilder builder)
    {
        // 게임 서비스 등록
        builder.Register<ICombatSystem, CombatSystem>(Lifetime.Scoped);
        builder.RegisterEntryPoint<GamePresenter>();
    }

    async UniTaskVoid Start()
    {
        // Additive UI 씬 로드 + VContainer 스코프 연결
        using (LifetimeScope.EnqueueParent(this))
        {
            _uiSceneHandle = Addressables.LoadSceneAsync(
                "UIScene", LoadSceneMode.Additive);
            await _uiSceneHandle;
        }
    }

    protected override void OnDestroy()
    {
        if (_uiSceneHandle.IsValid())
            Addressables.UnloadSceneAsync(_uiSceneHandle);
        base.OnDestroy();
    }
}
```

---

## 에셋 캐시 관리자

### 스코프 기반 에셋 캐시

LifetimeScope 수명에 맞춰 에셋을 자동 관리한다.

```csharp
public class ScopedAssetCache : IDisposable
{
    private readonly Dictionary<string, AsyncOperationHandle> _cache = new();

    public async UniTask<T> GetOrLoadAsync<T>(string address, CancellationToken ct)
    {
        if (_cache.TryGetValue(address, out var existing))
        {
            return (T)existing.Result;
        }

        var handle = Addressables.LoadAssetAsync<T>(address);
        await handle.WithCancellation(ct);

        if (handle.Status != AsyncOperationStatus.Succeeded)
        {
            Addressables.Release(handle);
            throw new System.Exception($"로드 실패: {address}");
        }

        _cache[address] = handle;
        return handle.Result;
    }

    public void Dispose()
    {
        foreach (var handle in _cache.Values)
        {
            if (handle.IsValid())
                Addressables.Release(handle);
        }
        _cache.Clear();
    }
}
```

등록:

```csharp
builder.Register<ScopedAssetCache>(Lifetime.Scoped);
```

---

## 에러 처리

### 로드 실패 대응

```csharp
public async UniTask<T> SafeLoadAsync<T>(string address, CancellationToken ct)
{
    AsyncOperationHandle<T> handle = default;

    try
    {
        handle = Addressables.LoadAssetAsync<T>(address);
        await handle.WithCancellation(ct);

        if (handle.Status == AsyncOperationStatus.Succeeded)
            return handle.Result;

        Debug.LogError($"에셋 로드 실패: {address}");
        return default;
    }
    catch (OperationCanceledException)
    {
        // 취소 시 핸들 릴리스
        if (handle.IsValid())
            Addressables.Release(handle);
        throw;
    }
    catch (Exception ex)
    {
        Debug.LogException(ex);
        if (handle.IsValid())
            Addressables.Release(handle);
        return default;
    }
}
```

### 부분 실패 처리 (LoadAssetsAsync)

```csharp
var handle = Addressables.LoadAssetsAsync<GameObject>(
    "enemy",
    addressable =>
    {
        if (addressable != null)
            ProcessEnemy(addressable);
    },
    releaseDependenciesOnFailure: false);  // 실패해도 로드된 것은 유지

await handle.WithCancellation(ct);

// 일부 실패 시 handle.Status == Failed이지만
// Result에는 성공한 에셋들이 포함되어 있다 (실패 항목은 null)
foreach (var asset in handle.Result)
{
    if (asset != null)
        UseAsset(asset);
}
```

---

## AsyncOperationHandle 패턴

### UniTask 변환 (권장)

```csharp
// WithCancellation (취소 지원)
var prefab = await Addressables.LoadAssetAsync<GameObject>("enemy")
    .WithCancellation(ct);

// ToUniTask (진행률 지원)
var prefab = await Addressables.LoadAssetAsync<GameObject>("enemy")
    .ToUniTask(progress: Progress.Create<float>(p => Debug.Log(p)),
               cancellationToken: ct);
```

### 코루틴 (레거시)

```csharp
IEnumerator LoadCoroutine()
{
    var handle = Addressables.LoadAssetAsync<GameObject>("enemy");
    yield return handle;

    if (handle.Status == AsyncOperationStatus.Succeeded)
        Instantiate(handle.Result);

    Addressables.Release(handle);
}
```

### 콜백 (이벤트 기반)

```csharp
void Start()
{
    var handle = Addressables.LoadAssetAsync<GameObject>("enemy");
    handle.Completed += OnLoadCompleted;
}

void OnLoadCompleted(AsyncOperationHandle<GameObject> handle)
{
    if (handle.Status == AsyncOperationStatus.Succeeded)
        Instantiate(handle.Result);
}
```
