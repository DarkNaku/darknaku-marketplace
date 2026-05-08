---
name: unity-zlinq
description: >
  ZLinq(제로 할당 LINQ)를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 ValueEnumerable 패턴, GameObject 트리 순회, GC 최적화 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - ZLinq, AsValueEnumerable, ValueEnumerable 언급 시
  - LINQ to Tree (Ancestors, Children, Descendants) 관련 작업
  - OfComponent, Transform/GameObject 계층 탐색 작업
  - GC 할당 최소화를 위한 LINQ 최적화 작업
  - NativeArray, NativeList 등 Unity Native Collection LINQ 작업

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - ZLinq를 사용하지 않는 프로젝트
  - GC 할당이 문제되지 않는 에디터 전용 코드
  - 단순 for/foreach로 충분한 반복
user-invocable: false
---

# Unity ZLinq Skill

## 핵심 원칙

> **런타임 LINQ는 ZLinq를 사용한다.**
> 표준 `System.Linq`는 GC 할당이 발생한다. 런타임 코드에서는 `AsValueEnumerable()`을 사용하여 제로 할당 LINQ 체인을 작성한다.

> **GameObject 트리 탐색에는 LINQ to Tree를 사용한다.**
> `GetComponentsInChildren`, 재귀 탐색 대신 `Descendants()`, `Children()`, `OfComponent<T>()`를 사용한다.

공식 저장소: https://github.com/Cysharp/ZLinq

---

## 기본 사용법

### AsValueEnumerable()

컬렉션을 제로 할당 LINQ 체인으로 변환한다.

```csharp
using ZLinq;

// 배열
int[] numbers = { 1, 2, 3, 4, 5 };
var result = numbers
    .AsValueEnumerable()
    .Where(x => x % 2 == 0)
    .Select(x => x * 3)
    .ToArray();

// List<T>
List<Enemy> enemies = GetEnemies();
var aliveEnemies = enemies
    .AsValueEnumerable()
    .Where(e => e.IsAlive)
    .ToArray();

// IEnumerable<T> (내부가 배열/리스트면 자동 최적화)
IEnumerable<int> source = GetNumbers();
foreach (var item in source.AsValueEnumerable().Where(x => x > 0))
{
    // 제로 할당
}
```

### 지원 컬렉션

| 컬렉션 | 제로 할당 |
|---|---|
| `T[]` | O |
| `List<T>` | O |
| `ArraySegment<T>` | O |
| `Memory<T>` / `ReadOnlyMemory<T>` | O |
| `Dictionary<TKey, TValue>` | O |
| `Queue<T>` / `Stack<T>` | O |
| `LinkedList<T>` | O |
| `HashSet<T>` | O |
| `ImmutableArray<T>` | O |
| `NativeArray<T>` / `NativeList<T>` | O (설정 필요) |
| `IEnumerable<T>` | 내부 타입에 따라 최적화 |

---

## LINQ to Tree (GameObject/Transform)

### 축(Axis) 메서드

Transform/GameObject 계층을 LINQ으로 탐색한다.

```csharp
using ZLinq;

// 조상 (부모 → 루트 방향)
foreach (var ancestor in transform.Ancestors())
    Debug.Log(ancestor.name);

// 자식 (직계 자식만)
foreach (var child in transform.Children())
    Debug.Log(child.name);

// 자손 (모든 하위 계층, 재귀)
foreach (var desc in transform.Descendants())
    Debug.Log(desc.name);

// 자신 포함 변형
foreach (var item in transform.ChildrenAndSelf())
    Debug.Log(item.name);

foreach (var item in transform.DescendantsAndSelf())
    Debug.Log(item.name);

foreach (var item in transform.AncestorsAndSelf())
    Debug.Log(item.name);

// 형제 (같은 부모의 자식들)
foreach (var sibling in transform.BeforeSelf())
    Debug.Log(sibling.name);  // 자신보다 앞의 형제

foreach (var sibling in transform.AfterSelf())
    Debug.Log(sibling.name);  // 자신보다 뒤의 형제
```

### 축 메서드 요약

| 메서드 | 탐색 범위 | 자신 포함 |
|---|---|---|
| `Ancestors()` | 부모 → 루트 | X |
| `AncestorsAndSelf()` | 자신 → 부모 → 루트 | O |
| `Children()` | 직계 자식 | X |
| `ChildrenAndSelf()` | 자신 + 직계 자식 | O |
| `Descendants()` | 모든 하위 계층 | X |
| `DescendantsAndSelf()` | 자신 + 모든 하위 계층 | O |
| `BeforeSelf()` | 자신보다 앞의 형제 | X |
| `AfterSelf()` | 자신보다 뒤의 형제 | X |

### 컴포넌트 필터링 (OfComponent)

`GetComponentsInChildren<T>()` 대신 사용한다. GC 할당이 없다.

```csharp
// 자손에서 특정 컴포넌트 찾기
var fooScripts = root.Descendants().OfComponent<FooScript>();

foreach (var foo in fooScripts)
{
    foo.DoSomething();
}

// 자신 포함
var allRenderers = root.DescendantsAndSelf().OfComponent<Renderer>();

// 자식만 (직계)
var childColliders = root.Children().OfComponent<Collider>();
```

### 조건 필터링

```csharp
// 태그로 필터
var enemies = root.Descendants()
    .Where(t => t.CompareTag("Enemy"));

// 활성 오브젝트만
var activeChildren = root.Children()
    .Where(t => t.gameObject.activeSelf);

// 레이어로 필터
var uiElements = root.Descendants()
    .Where(t => t.gameObject.layer == LayerMask.NameToLayer("UI"));

// 이름 패턴으로 찾기
var waypoints = root.Descendants()
    .Where(t => t.name.StartsWith("Waypoint_"))
    .Select(t => t.position)
    .ToArray();
```

### GetComponentsInChildren 대체 패턴

```csharp
// Before (GC 할당)
var renderers = transform.GetComponentsInChildren<Renderer>();

// After (제로 할당)
foreach (var renderer in transform.DescendantsAndSelf().OfComponent<Renderer>())
{
    renderer.enabled = false;
}

// Before
var firstEnemy = transform.GetComponentInChildren<EnemyAI>();

// After
var firstEnemy = transform.DescendantsAndSelf()
    .OfComponent<EnemyAI>()
    .FirstOrDefault();
```

---

## UI Toolkit 트리 탐색

VisualElement 계층도 동일한 축 메서드로 탐색한다.

```csharp
using ZLinq;

var root = document.rootVisualElement;

// 모든 버튼 찾기
var allButtons = root.Descendants().OfType<Button>();

// 텍스트가 비어있는 버튼 찾기
var emptyButtons = root.Descendants()
    .OfType<Button>()
    .Where(btn => string.IsNullOrEmpty(btn.text));

// 특정 클래스가 있는 요소
var highlighted = root.Descendants()
    .Where(ve => ve.ClassListContains("highlighted"));
```

---

## 표준 오퍼레이터

### 자주 사용하는 오퍼레이터

```csharp
var source = items.AsValueEnumerable();

// 필터링
source.Where(x => x.IsActive)
source.OfType<Derived>()
source.Distinct()
source.DistinctBy(x => x.Id)

// 변환
source.Select(x => x.Name)
source.SelectMany(x => x.Children)
source.Cast<Derived>()

// 정렬
source.OrderBy(x => x.Priority)
source.OrderByDescending(x => x.Score)
source.ThenBy(x => x.Name)

// 수량 제한
source.Take(5)
source.Skip(2)
source.TakeWhile(x => x.IsValid)
source.SkipWhile(x => x.IsPending)

// 집계
source.Count()
source.Any(x => x.IsAlive)
source.All(x => x.IsReady)
source.Sum(x => x.Score)
source.Average(x => x.Height)
source.Min(x => x.Distance)
source.Max(x => x.Priority)

// 단일 요소
source.First()
source.FirstOrDefault()
source.Last()
source.ElementAt(3)

// 결합
source.Concat(other.AsValueEnumerable())
source.Zip(other.AsValueEnumerable(), (a, b) => (a, b))
source.Union(other.AsValueEnumerable())
source.Except(other.AsValueEnumerable())
source.Intersect(other.AsValueEnumerable())

// 그룹
source.GroupBy(x => x.Category)
source.Join(other, x => x.Id, y => y.Id, (x, y) => new { x, y })

// 기타
source.Reverse()
source.Append(newItem)
source.Prepend(firstItem)
source.Aggregate((acc, x) => acc + x)
```

### 결과 수집

```csharp
// 배열로 변환
var array = source.ToArray();

// 리스트로 변환
var list = source.ToList();

// 딕셔너리로 변환
var dict = source.ToDictionary(x => x.Id);

// 해시셋으로 변환
var set = source.ToHashSet();

// 문자열 결합 (String.Join 대체)
var text = source.Select(x => x.Name).JoinToString(", ");

// ArrayPool에서 빌려오기 (고성능)
using var pooled = source.ToArrayPool();
ProcessSpan(pooled.Span);
```

---

## Native Collection 지원

NativeArray, NativeList 등 Unity Native Collection을 ZLinq으로 사용하려면 어셈블리에 어트리뷰트를 추가한다.

```csharp
[assembly: ZLinqDropInExternalExtension("ZLinq",
    "Unity.Collections.NativeArray`1", "ZLinq.Linq.FromNativeArray`1")]
[assembly: ZLinqDropInExternalExtension("ZLinq",
    "Unity.Collections.NativeList`1", "ZLinq.Linq.FromNativeList`1")]
```

사용:

```csharp
NativeArray<float> positions = new NativeArray<float>(1000, Allocator.Temp);

var filtered = positions
    .AsValueEnumerable()
    .Where(p => p > 0f)
    .ToArray();

positions.Dispose();
```

---

## Drop-In 소스 생성기

표준 LINQ 호출을 자동으로 ZLinq으로 대체한다.

```csharp
// 어셈블리 레벨 어트리뷰트
[assembly: ZLinqDropIn("ZLinq", DropInGenerateTypes.Array | DropInGenerateTypes.List)]
```

| DropInGenerateTypes | 대상 |
|---|---|
| `Array` | `T[]` |
| `List` | `List<T>` |
| `Span` | `Span<T>` |
| `Memory` | `Memory<T>` |
| `Enumerable` | `IEnumerable<T>` |
| `Collection` | Array + Span + Memory + List |
| `Everything` | 모든 타입 |

설정 후 기존 `.Where()`, `.Select()` 등이 자동으로 ZLinq 버전으로 교체된다.

---

## 주의사항 및 제약

### ValueEnumerable은 재할당 불가

체인의 각 단계가 다른 타입을 반환하므로 변수에 재할당할 수 없다.

```csharp
// 컴파일 에러
var result = source.AsValueEnumerable().Where(x => x > 5);
result = result.Select(x => x * 2);  // 타입 불일치

// 올바른 방법: 한 번에 체인 작성
var result = source.AsValueEnumerable()
    .Where(x => x > 5)
    .Select(x => x * 2)
    .ToArray();
```

### yield/await와 함께 사용 불가

ValueEnumerable은 ref struct이므로 yield return이나 await 경계를 넘을 수 없다.

```csharp
// 불가: yield return과 함께 사용
IEnumerable<int> Bad()
{
    var ve = source.AsValueEnumerable();  // ref struct
    yield return 1;  // 컴파일 에러
}

// 올바른 방법: ToArray()로 먼저 수집
async UniTask GoodAsync()
{
    var result = source.AsValueEnumerable()
        .Where(x => x > 0)
        .ToArray();  // 여기서 수집

    await UniTask.Yield();
    // result(배열)는 await 이후에도 사용 가능
}
```

### 작은 컬렉션에서의 오버헤드

깊은 체인은 struct 복사 비용이 있다. 요소가 매우 적은 경우(< 10개) 일반 for 루프가 더 빠를 수 있다.

### String.Join 비호환

`String.Join`에 ValueEnumerable을 전달하면 `params object[]` 오버로드가 선택된다.
`JoinToString()`을 사용한다.

```csharp
// 하지 말 것
string.Join(", ", source.AsValueEnumerable().Select(x => x.Name));

// 올바른 방법
source.AsValueEnumerable().Select(x => x.Name).JoinToString(", ");
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| 런타임에서 `System.Linq` 사용 | `AsValueEnumerable()` + ZLinq |
| `GetComponentsInChildren<T>()` | `DescendantsAndSelf().OfComponent<T>()` |
| `String.Join`에 ValueEnumerable 전달 | `.JoinToString(separator)` |
| ValueEnumerable을 변수에 재할당 | 한 번에 체인 작성 후 `ToArray()` |
| ValueEnumerable을 await 이후 사용 | await 전에 `ToArray()`로 수집 |
| 요소 3개에 긴 체인 작성 | 짧은 반복은 for 루프 |
| `foreach`에서 `System.Linq.Where` 사용 | `AsValueEnumerable().Where()` |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| LINQ to Tree 축 메서드 상세, 커스텀 트리 탐색 구현 | `references/tree-traversal.md` |
