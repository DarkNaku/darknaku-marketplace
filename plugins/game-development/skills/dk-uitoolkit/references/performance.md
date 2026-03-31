# Unity UI Toolkit — 성능 최적화 참조

## 핵심 원칙

UI Toolkit은 내부적으로 **Retained Mode** 렌더링을 사용한다.  
즉, 변경된 부분만 다시 그리므로 불필요한 레이아웃 재계산을 최소화하는 게 핵심이다.

---

## Q<>() 쿼리 최적화

```csharp
// ❌ 나쁜 예: Update / 매 프레임 호출
private void Update()
{
    _uiDocument.rootVisualElement.Q<Label>("score").text = score.ToString();
}

// ✅ 좋은 예: Awake에서 캐싱
private Label _scoreLabel;

private void Awake()
{
    _scoreLabel = _uiDocument.rootVisualElement.Q<Label>("score");
}

private void SetScore(int score)
{
    _scoreLabel.text = score.ToString();
}
```

---

## 스타일 변경 최적화

```csharp
// ❌ 나쁜 예: 인라인 스타일 다중 변경 (레이아웃 재계산 여러 번)
element.style.width = 100;
element.style.height = 50;
element.style.backgroundColor = Color.red;

// ✅ 좋은 예: USS 클래스 토글 (배치 처리)
element.AddToClassList("state--active");
element.RemoveFromClassList("state--inactive");

// ✅ 또는 EnableInClassList 사용
element.EnableInClassList("state--active", isActive);
```

---

## ListView 성능 (대량 아이템)

```csharp
// ListView는 가상화(Virtualization)를 지원한다
// 반드시 fixedItemHeight 또는 virtualizationMethod 설정

var listView = root.Q<ListView>();

// 고정 높이 아이템 (최고 성능)
listView.fixedItemHeight = 60f;
listView.virtualizationMethod = CollectionVirtualizationMethod.FixedHeight;

// 가변 높이 아이템 (약간 느림)
listView.virtualizationMethod = CollectionVirtualizationMethod.DynamicHeight;

// makeItem은 재사용되므로 new() 비용 최소화
listView.makeItem = () =>
{
    var item = new VisualElement();
    item.Add(new Label { name = "item-label" });
    return item;
};

// bindItem에서는 데이터만 업데이트
listView.bindItem = (element, index) =>
{
    element.Q<Label>("item-label").text = _items[index].Name;
};
```

---

## 동적 UI 풀링

```csharp
// 팝업/아이템 등 자주 생성/파괴되는 경우 풀링 권장
public class VisualElementPool<T> where T : VisualElement, new()
{
    private readonly Stack<T> _pool = new();

    public T Get()
        => _pool.Count > 0 ? _pool.Pop() : new T();

    public void Return(T element)
    {
        element.RemoveFromHierarchy();
        _pool.Push(element);
    }
}
```

---

## 레이아웃 재계산 최소화

```csharp
// 여러 요소를 한 번에 추가할 때 부모를 먼저 숨기기
container.style.display = DisplayStyle.None;
foreach (var data in dataList)
{
    var item = CreateItem(data);
    container.Add(item);
}
container.style.display = DisplayStyle.Flex;

// schedule로 다음 프레임에 처리
element.schedule.Execute(() => DoExpensiveLayout()).StartingIn(1);
```

---

## USS Transition vs C# 애니메이션

```css
/* USS Transition — 권장 (UI Toolkit 내부 최적화) */
.panel {
    opacity: 0;
    transition: opacity 0.3s ease-in-out;
}

.panel--visible {
    opacity: 1;
}
```

```csharp
// C# DOTween 애니메이션 — 복잡한 시퀀스에만 사용
// UI Toolkit은 DOTween 직접 지원 안 함, 수동 스케줄 필요
element.schedule.Execute(() =>
{
    var t = (Time.time - startTime) / duration;
    element.style.opacity = Mathf.Lerp(0f, 1f, t);
}).Until(() => Time.time >= startTime + duration);
```

---

## 메모리 관리

```csharp
// 이벤트 콜백 반드시 해제
private void OnDestroy()
{
    // clicked 이벤트
    _button.clicked -= OnClicked;

    // RegisterCallback 방식
    _element.UnregisterCallback<ClickEvent>(OnClick);

    // R3 Disposable
    _disposables.Dispose();
}

// 씬 전환 시 UIDocument는 자동 정리되지만
// 외부 참조(static, 싱글톤 등)는 수동으로 null 처리
```

---

## 프로파일링

- Unity Profiler → **UI Toolkit** 카테고리 확인
- `Layout` 단계 비용이 높으면 불필요한 스타일 변경 확인
- `Repaint` 단계 비용이 높으면 투명도/블렌딩 과다 사용 의심
- **UIElements Debugger** (Window > UI Toolkit > Debugger) 로 레이아웃 박스 확인
