# LINQ to Tree 상세

## 축(Axis) 개념

ZLinq의 LINQ to Tree는 XPath/XLinq의 축 개념을 차용한다.
트리 구조의 노드에서 특정 방향으로 관련 노드를 탐색한다.

```
         Root
        /    \
       A      B          ← Children of Root
      / \      \
     C   D      E        ← Descendants of Root
    /
   F                     ← Descendants of Root, Children of C
```

---

## Transform/GameObject 트리

### Ancestors (조상)

현재 노드의 부모에서 루트까지 올라간다.

```csharp
// D의 조상: A → Root
foreach (var ancestor in nodeD.Ancestors())
    Debug.Log(ancestor.name);

// 자신 포함: D → A → Root
foreach (var ancestor in nodeD.AncestorsAndSelf())
    Debug.Log(ancestor.name);
```

**활용: 특정 부모 찾기**

```csharp
// 가장 가까운 Canvas 찾기
var canvas = transform.Ancestors()
    .OfComponent<Canvas>()
    .FirstOrDefault();

// 특정 태그의 부모까지 올라가기
var room = transform.Ancestors()
    .FirstOrDefault(t => t.CompareTag("Room"));
```

### Children (자식)

직계 자식만 탐색한다. 손자 이하는 포함하지 않는다.

```csharp
// Root의 자식: A, B
foreach (var child in root.Children())
    Debug.Log(child.name);
```

**활용: 자식 정리**

```csharp
// 비활성 자식 제거
var inactiveChildren = transform.Children()
    .Where(t => !t.gameObject.activeSelf)
    .ToArray();  // 순회 중 삭제 방지를 위해 먼저 수집

foreach (var child in inactiveChildren)
    Object.Destroy(child.gameObject);
```

### Descendants (자손)

모든 하위 계층을 재귀적으로 탐색한다.

```csharp
// Root의 자손: A, C, F, D, B, E (깊이 우선)
foreach (var desc in root.Descendants())
    Debug.Log(desc.name);
```

**활용: 전체 하위 컴포넌트 수집**

```csharp
// 모든 하위 Renderer 비활성화
foreach (var renderer in root.DescendantsAndSelf().OfComponent<Renderer>())
    renderer.enabled = false;

// 특정 이름 패턴의 노드 찾기
var spawnPoints = level.Descendants()
    .Where(t => t.name.StartsWith("SpawnPoint_"))
    .Select(t => t.position)
    .ToArray();

// 특정 깊이까지만 탐색 (Take 활용)
var shallow = root.Descendants()
    .TakeWhile(t => t.GetSiblingIndex() < 10)
    .ToArray();
```

### BeforeSelf / AfterSelf (형제)

같은 부모의 자식들 중 자신보다 앞/뒤의 형제를 탐색한다.

```csharp
//   Parent
//   ├── A      ← BeforeSelf of C
//   ├── B      ← BeforeSelf of C
//   ├── C      ← 기준
//   ├── D      ← AfterSelf of C
//   └── E      ← AfterSelf of C

foreach (var before in nodeC.BeforeSelf())
    Debug.Log(before.name);  // A, B

foreach (var after in nodeC.AfterSelf())
    Debug.Log(after.name);  // D, E
```

---

## OfComponent vs GetComponent 비교

### 단일 컴포넌트 탐색

```csharp
// Before: GC 할당 발생
var enemy = transform.GetComponentInChildren<EnemyAI>();

// After: 제로 할당
var enemy = transform.DescendantsAndSelf()
    .OfComponent<EnemyAI>()
    .FirstOrDefault();
```

### 다중 컴포넌트 수집

```csharp
// Before: GC 할당 (배열 생성)
Renderer[] renderers = transform.GetComponentsInChildren<Renderer>();

// After: 제로 할당 순회
foreach (var renderer in transform.DescendantsAndSelf().OfComponent<Renderer>())
{
    renderer.material.color = Color.red;
}

// 배열이 필요한 경우에만 ToArray()
var rendererArray = transform.DescendantsAndSelf()
    .OfComponent<Renderer>()
    .ToArray();
```

### 조건 + 컴포넌트 결합

```csharp
// 활성 오브젝트의 Collider만
var activeColliders = transform.DescendantsAndSelf()
    .Where(t => t.gameObject.activeSelf)
    .OfComponent<Collider>();

// 특정 레이어의 Renderer
int targetLayer = LayerMask.NameToLayer("Interactable");
var interactableRenderers = transform.Descendants()
    .Where(t => t.gameObject.layer == targetLayer)
    .OfComponent<Renderer>();
```

---

## UI Toolkit VisualElement 트리

VisualElement 계층도 동일한 축 메서드로 탐색한다.

```csharp
var root = document.rootVisualElement;

// 모든 자손 중 Button 타입
var buttons = root.Descendants()
    .OfType<Button>()
    .ToArray();

// 특정 USS 클래스를 가진 요소
var highlightedLabels = root.Descendants()
    .OfType<Label>()
    .Where(label => label.ClassListContains("highlighted"));

// 특정 이름의 요소 찾기 (Q<T> 대체)
var scoreLabel = root.Descendants()
    .OfType<Label>()
    .FirstOrDefault(l => l.name == "score-label");

// 비어있는 텍스트를 가진 버튼 찾기
var emptyButtons = root.Descendants()
    .OfType<Button>()
    .Where(btn => string.IsNullOrEmpty(btn.text));
```

---

## 실전 패턴

### 오브젝트 풀에서 비활성 객체 찾기

```csharp
public Transform FindInactiveInPool(Transform poolRoot)
{
    return poolRoot.Children()
        .FirstOrDefault(t => !t.gameObject.activeSelf);
}
```

### 가장 가까운 적 찾기

```csharp
public EnemyAI FindNearestEnemy(Transform root, Vector3 position)
{
    return root.Descendants()
        .OfComponent<EnemyAI>()
        .Where(e => e.IsAlive)
        .OrderBy(e => Vector3.Distance(e.transform.position, position))
        .FirstOrDefault();
}
```

### 계층 경로 생성

```csharp
public string GetHierarchyPath(Transform target)
{
    return target.AncestorsAndSelf()
        .Reverse()
        .Select(t => t.name)
        .JoinToString("/");
}
```

### 특정 컴포넌트가 있는 가장 가까운 조상

```csharp
public T FindInAncestors<T>(Transform start) where T : Component
{
    return start.Ancestors()
        .OfComponent<T>()
        .FirstOrDefault();
}
```
