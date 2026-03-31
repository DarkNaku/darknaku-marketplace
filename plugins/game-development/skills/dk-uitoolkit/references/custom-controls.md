# Unity UI Toolkit — 커스텀 컨트롤 참조

## VisualElement 상속 커스텀 컨트롤

재사용 가능한 UI 컴포넌트가 필요할 때 `VisualElement`를 상속한다.

### 기본 커스텀 컨트롤

```csharp
// HealthBar.cs — 재사용 가능한 커스텀 컨트롤
using UnityEngine.UIElements;

public class HealthBar : VisualElement
{
    // UxmlFactory: UXML에서 <HealthBar /> 태그로 사용 가능하게 등록
    public new class UxmlFactory : UxmlFactory<HealthBar, UxmlTraits> { }

    public new class UxmlTraits : VisualElement.UxmlTraits
    {
        // UXML 속성 정의
        private readonly UxmlFloatAttributeDescription _maxValue =
            new() { name = "max-value", defaultValue = 100f };
        private readonly UxmlColorAttributeDescription _fillColor =
            new() { name = "fill-color", defaultValue = Color.green };

        public override void Init(VisualElement ve, IUxmlAttributes bag, CreationContext cc)
        {
            base.Init(ve, bag, cc);
            var bar = (HealthBar)ve;
            bar.MaxValue = _maxValue.GetValueFromBag(bag, cc);
            bar.FillColor = _fillColor.GetValueFromBag(bag, cc);
        }
    }

    private readonly VisualElement _fill;

    public float MaxValue { get; set; } = 100f;
    public Color FillColor { get; set; } = Color.green;

    private float _currentValue;
    public float CurrentValue
    {
        get => _currentValue;
        set
        {
            _currentValue = Mathf.Clamp(value, 0f, MaxValue);
            _fill.style.width = Length.Percent((_currentValue / MaxValue) * 100f);
        }
    }

    public HealthBar()
    {
        // 기본 구조 생성
        AddToClassList("health-bar");

        var background = new VisualElement();
        background.AddToClassList("health-bar__background");
        Add(background);

        _fill = new VisualElement();
        _fill.AddToClassList("health-bar__fill");
        background.Add(_fill);
    }
}
```

```xml
<!-- UXML에서 사용 -->
<ui:UXML xmlns:ui="UnityEngine.UIElements" xmlns:custom="Assembly-CSharp.Editor.UIElements">
    <HealthBar name="player-health" max-value="100" />
</ui:UXML>
```

### Unity 6+ UxmlElement (신규 API)

```csharp
// Unity 6부터는 source generator 방식 권장
[UxmlElement]
public partial class HealthBar : VisualElement
{
    [UxmlAttribute]
    public float MaxValue { get; set; } = 100f;

    // ... 구현
}
```

---

## 커스텀 이벤트

```csharp
// 커스텀 이벤트 정의
public class ValueChangedEvent : EventBase<ValueChangedEvent>
{
    public float NewValue { get; private set; }

    public static ValueChangedEvent GetPooled(float value)
    {
        var evt = GetPooled();
        evt.NewValue = value;
        return evt;
    }
}

// 이벤트 발송
private void OnFillChanged(float value)
{
    using var evt = ValueChangedEvent.GetPooled(value);
    evt.target = this;
    SendEvent(evt);
}

// 이벤트 구독 (외부)
healthBar.RegisterCallback<ValueChangedEvent>(e =>
    Debug.Log($"Health changed: {e.NewValue}"));
```

---

## Manipulator (입력 처리)

```csharp
// 드래그 가능한 슬라이더 Manipulator 예시
public class DragManipulator : MouseManipulator
{
    private bool _isDragging;

    public DragManipulator()
    {
        activators.Add(new ManipulatorActivationFilter
            { button = MouseButton.LeftMouse });
    }

    protected override void RegisterCallbacksOnTarget()
    {
        target.RegisterCallback<MouseDownEvent>(OnMouseDown);
        target.RegisterCallback<MouseMoveEvent>(OnMouseMove);
        target.RegisterCallback<MouseUpEvent>(OnMouseUp);
    }

    protected override void UnregisterCallbacksFromTarget()
    {
        target.UnregisterCallback<MouseDownEvent>(OnMouseDown);
        target.UnregisterCallback<MouseMoveEvent>(OnMouseMove);
        target.UnregisterCallback<MouseUpEvent>(OnMouseUp);
    }

    private void OnMouseDown(MouseDownEvent e)
    {
        if (!CanStartManipulation(e)) return;
        _isDragging = true;
        target.CaptureMouse();
        e.StopPropagation();
    }

    private void OnMouseMove(MouseMoveEvent e)
    {
        if (!_isDragging) return;
        target.style.left = target.resolvedStyle.left + e.mouseDelta.x;
        e.StopPropagation();
    }

    private void OnMouseUp(MouseUpEvent e)
    {
        if (!_isDragging || !CanStopManipulation(e)) return;
        _isDragging = false;
        target.ReleaseMouse();
        e.StopPropagation();
    }
}

// 사용
myElement.AddManipulator(new DragManipulator());
```

---

## 참고 예제 (공식 저장소)
- `create-custom-controls/` : 커스텀 컨트롤 전체 예제
- URL: https://github.com/Unity-Technologies/ui-toolkit-manual-code-examples
