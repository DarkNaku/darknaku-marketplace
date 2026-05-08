---
name: unity-addressables
description: >
  Addressables(에셋 관리 시스템)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 에셋 로딩/해제 패턴, AssetReference 사용, 메모리 관리 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - Addressables, AssetReference, AsyncOperationHandle 언급 시
  - LoadAssetAsync, InstantiateAsync, LoadSceneAsync 관련 작업
  - 에셋 번들, 리소스 로딩, 동적 에셋 관리 작업
  - Addressables.Release, ReleaseAsset, ReleaseInstance 관련 작업
  - 에셋 주소, 라벨 기반 에셋 로딩

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - Addressables를 사용하지 않는 프로젝트
  - Resources.Load만 사용하는 프로젝트
  - 에디터 전용 에셋 로딩 (AssetDatabase)
user-invocable: false
---

# Unity Addressables Skill

## 핵심 원칙

> **모든 로드에는 대응하는 릴리스가 있어야 한다.**
> `LoadAssetAsync` → `Release`, `InstantiateAsync` → `ReleaseInstance`, `LoadSceneAsync` → `UnloadSceneAsync`.
> 릴리스 누락은 메모리 누수의 직접적 원인이다.

> **AssetReference를 기본으로 사용한다.**
> 문자열 주소 대신 Inspector에서 할당하는 `AssetReference`를 사용하여 타입 안전성과 리팩터링 용이성을 확보한다.

> **UniTask와 함께 사용한다.**
> `AsyncOperationHandle.Task` 대신 `.WithCancellation(ct)` 또는 `.ToUniTask()`로 UniTask 패턴을 따른다.

공식 문서: https://docs.unity3d.com/Packages/com.unity.addressables@2.11/manual/AddressableAssetsOverview.html

---

## 핵심 개념

### 주소 (Address)

에셋의 물리적 경로가 아닌 논리적 이름으로 에셋을 참조한다.

```
물리 경로: Assets/Prefabs/Enemies/Boss_Dragon.prefab
주소:      boss_dragon
```

### 그룹 (Group)

Addressable 에셋은 그룹에 속한다. 그룹 설정이 AssetBundle 패키징 방식을 결정한다.

### 라벨 (Label)

에셋에 태그를 붙여 일괄 로딩에 사용한다. 하나의 에셋에 여러 라벨을 할당할 수 있다.

```
에셋: boss_dragon  → 라벨: [enemy, boss, fire]
에셋: goblin       → 라벨: [enemy, normal]
```

### 레퍼런스 카운팅

Addressables는 참조 카운트로 메모리를 관리한다. 로드 시 증가, 릴리스 시 감소. 카운트가 0이 되면 AssetBundle과 함께 언로드된다.

---

## AssetReference (Inspector 할당)

### 기본 사용법

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class AssetLoader : MonoBehaviour
{
    [SerializeField] private AssetReference _prefabRef;

    private AsyncOperationHandle<GameObject> _handle;

    async UniTaskVoid Start()
    {
        _handle = _prefabRef.LoadAssetAsync<GameObject>();
        await _handle.WithCancellation(destroyCancellationToken);

        if (_handle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(_handle.Result, transform);
        }
    }

    void OnDestroy()
    {
        if (_handle.IsValid())
            Addressables.Release(_handle);
    }
}
```

### 타입 제한 AssetReference

Inspector에서 잘못된 타입의 에셋 할당을 방지한다.

```csharp
[SerializeField] private AssetReferenceGameObject _prefabRef;    // GameObject만
[SerializeField] private AssetReferenceTexture2D _textureRef;    // Texture2D만
[SerializeField] private AssetReferenceSprite _spriteRef;        // Sprite만
[SerializeField] private AssetReferenceAtlasedSprite _atlasRef;  // Atlas Sprite만
[SerializeField] private AssetReferenceT<AudioClip> _audioRef;   // 커스텀 타입
```

### AssetReference 릴리스

```csharp
// LoadAssetAsync로 로드한 경우
_prefabRef.ReleaseAsset();

// InstantiateAsync로 생성한 경우
_prefabRef.ReleaseInstance(instanceObj);
```

---

## 주소로 로딩

### 단일 에셋 로딩

```csharp
public async UniTask<T> LoadAssetAsync<T>(string address, CancellationToken ct)
{
    var handle = Addressables.LoadAssetAsync<T>(address);
    await handle.WithCancellation(ct);

    if (handle.Status == AsyncOperationStatus.Succeeded)
        return handle.Result;

    throw new System.Exception($"에셋 로드 실패: {address}");
}
```

### 여러 에셋 로딩 (라벨)

```csharp
public async UniTask<IList<GameObject>> LoadByLabelAsync(
    string label, CancellationToken ct)
{
    var handle = Addressables.LoadAssetsAsync<GameObject>(
        label,
        addressable =>
        {
            // 개별 에셋 로드 완료 시 콜백
            Debug.Log($"로드됨: {addressable.name}");
        });

    await handle.WithCancellation(ct);
    return handle.Result;
}
```

### 여러 키로 필터 로딩 (MergeMode)

```csharp
var keys = new List<string> { "enemy", "boss" };

// Union: enemy OR boss 라벨이 있는 모든 에셋
var handle = Addressables.LoadAssetsAsync<GameObject>(
    keys, null, Addressables.MergeMode.Union);

// Intersection: enemy AND boss 라벨이 모두 있는 에셋만
var handle = Addressables.LoadAssetsAsync<GameObject>(
    keys, null, Addressables.MergeMode.Intersection);
```

### 서브에셋 로딩 (스프라이트 시트)

```csharp
// 스프라이트 시트에서 특정 스프라이트
var sprite = await Addressables.LoadAssetAsync<Sprite>(
    "MySpriteSheet[SpriteName]").WithCancellation(ct);

// 스프라이트 시트의 모든 스프라이트
var sprites = await Addressables.LoadAssetAsync<IList<Sprite>>(
    "MySpriteSheet").WithCancellation(ct);
```

---

## InstantiateAsync (프리팹 생성)

### 기본 패턴

```csharp
var handle = Addressables.InstantiateAsync(address, position, rotation, parent);
await handle.WithCancellation(ct);

var instance = handle.Result;

// 인스턴스 해제 (Destroy + Release)
Addressables.ReleaseInstance(instance);
```

### InstantiateAsync vs LoadAssetAsync + Instantiate

| | InstantiateAsync | LoadAssetAsync + Instantiate |
|---|---|---|
| 참조 카운트 | 인스턴스당 1 증가 | 로드 1회만 증가 |
| 해제 | `ReleaseInstance(instance)` | `Release(handle)` |
| 적합한 상황 | 인스턴스 수가 적을 때 | 같은 프리팹을 많이 생성할 때 |
| 추적 | 자동 추적 (trackHandle) | 수동 관리 |

**같은 프리팹을 여러 개 생성할 때:**

```csharp
// 권장: 한 번 로드 후 여러 번 Instantiate
private AsyncOperationHandle<GameObject> _prefabHandle;

public async UniTask InitAsync(CancellationToken ct)
{
    _prefabHandle = Addressables.LoadAssetAsync<GameObject>("enemy_goblin");
    await _prefabHandle.WithCancellation(ct);
}

public GameObject SpawnEnemy(Vector3 position)
{
    return Object.Instantiate(_prefabHandle.Result, position, Quaternion.identity);
}

public void Cleanup()
{
    // Instantiate로 만든 인스턴스는 일반 Destroy
    // 원본 핸들만 Release
    Addressables.Release(_prefabHandle);
}
```

---

## 씬 로딩

### Additive 씬 로딩

```csharp
private AsyncOperationHandle<SceneInstance> _sceneHandle;

public async UniTask LoadSceneAdditiveAsync(string sceneAddress, CancellationToken ct)
{
    _sceneHandle = Addressables.LoadSceneAsync(
        sceneAddress, LoadSceneMode.Additive);
    await _sceneHandle.WithCancellation(ct);
}

public async UniTask UnloadSceneAsync()
{
    if (_sceneHandle.IsValid())
    {
        await Addressables.UnloadSceneAsync(_sceneHandle);
    }
}
```

### 활성화 지연 (로딩 화면)

```csharp
public async UniTask LoadWithProgressAsync(string sceneAddress, CancellationToken ct)
{
    var handle = Addressables.LoadSceneAsync(
        sceneAddress, LoadSceneMode.Additive, activateOnLoad: false);

    // 진행률 표시
    while (!handle.IsDone)
    {
        _loadingView.SetProgress(handle.PercentComplete);
        await UniTask.Yield(ct);
    }

    // 준비 완료 후 활성화
    await handle.Result.ActivateAsync();
}
```

---

## 릴리스 (메모리 해제)

### 릴리스 규칙

| 로드 방법 | 릴리스 방법 |
|---|---|
| `Addressables.LoadAssetAsync` | `Addressables.Release(handle)` |
| `Addressables.LoadAssetsAsync` | `Addressables.Release(handle)` |
| `Addressables.InstantiateAsync` | `Addressables.ReleaseInstance(instance)` |
| `Addressables.LoadSceneAsync` | `Addressables.UnloadSceneAsync(handle)` |
| `assetRef.LoadAssetAsync` | `assetRef.ReleaseAsset()` |
| `assetRef.InstantiateAsync` | `assetRef.ReleaseInstance(instance)` |

### AsyncOperationHandle 유효성 확인

```csharp
if (_handle.IsValid())
{
    Addressables.Release(_handle);
}
```

### DontDestroyOnLoad 주의

`Object.DontDestroyOnLoad`과 `HideFlags.DontUnloadUnusedAsset`는 Addressable 에셋에 효과가 없다. 씬 전환 시 에셋을 유지하려면 핸들을 유지해야 한다.

---

## AssetBundle 메모리 고려사항

### 부분 언로드 불가

AssetBundle은 부분 언로드가 불가능하다. 번들 내 3개 에셋 중 1개만 릴리스해도 번들 전체가 메모리에 남는다.

```
Bundle A: [에셋1, 에셋2, 에셋3]
에셋1 Release → Bundle A 여전히 로드 (에셋2, 3이 사용 중)
에셋2 Release → Bundle A 여전히 로드 (에셋3이 사용 중)
에셋3 Release → Bundle A 언로드 (참조 카운트 0)
```

### 에셋 번동 방지

릴리스 직후 같은 에셋을 다시 로드하면 성능이 저하된다. 에셋은 가능한 오래 유지한다.

---

## VContainer 연동 패턴

### 에셋 로더 서비스

```csharp
public interface IAssetLoader
{
    UniTask<T> LoadAsync<T>(string address, CancellationToken ct);
    UniTask<T> LoadAsync<T>(AssetReference reference, CancellationToken ct);
    void Release<T>(AsyncOperationHandle<T> handle);
}

public class AddressableAssetLoader : IAssetLoader, IDisposable
{
    private readonly List<AsyncOperationHandle> _handles = new();

    public async UniTask<T> LoadAsync<T>(string address, CancellationToken ct)
    {
        var handle = Addressables.LoadAssetAsync<T>(address);
        _handles.Add(handle);
        await handle.WithCancellation(ct);

        if (handle.Status != AsyncOperationStatus.Succeeded)
            throw new System.Exception($"로드 실패: {address}");

        return handle.Result;
    }

    public async UniTask<T> LoadAsync<T>(AssetReference reference, CancellationToken ct)
    {
        var handle = reference.LoadAssetAsync<T>();
        _handles.Add(handle);
        await handle.WithCancellation(ct);
        return (T)handle.Result;
    }

    public void Release<T>(AsyncOperationHandle<T> handle)
    {
        if (handle.IsValid())
        {
            _handles.Remove(handle);
            Addressables.Release(handle);
        }
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

### LifetimeScope 등록

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IAssetLoader, AddressableAssetLoader>(Lifetime.Scoped);
    }
}
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `Release` 호출 누락 | 모든 로드에 대응하는 릴리스 보장 |
| 문자열 주소 하드코딩 | `AssetReference` Inspector 할당 |
| 같은 프리팹을 `InstantiateAsync`로 대량 생성 | `LoadAssetAsync` 1회 + `Object.Instantiate` N회 |
| `DontDestroyOnLoad`로 에셋 유지 시도 | 핸들을 DontDestroyOnLoad 오브젝트에서 관리 |
| 릴리스 직후 같은 에셋 재로드 반복 | 에셋을 가능한 오래 유지 |
| `AsyncOperationHandle`을 확인 없이 릴리스 | `handle.IsValid()` 확인 후 릴리스 |
| 로드 실패 시 에러 무시 | `handle.Status` 확인 후 예외 처리 |
| `async void`에서 Addressables 로딩 | `async UniTask`에서 `WithCancellation(ct)` |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| 에셋 로더 설계, 프리로딩, 오브젝트 풀 연동, 씬 전환 패턴 | `references/loading-patterns.md` |
