# 런타임 리바인딩

## 인터랙티브 리바인딩

사용자가 런타임에 직접 키를 변경할 수 있는 기능을 구현한다.

### 기본 패턴

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class RebindManager
{
    private InputActionRebindingExtensions.RebindingOperation _rebindOperation;

    public void StartRebind(InputAction action, int bindingIndex, System.Action onComplete)
    {
        action.Disable();

        _rebindOperation = action.PerformInteractiveRebinding(bindingIndex)
            .WithControlsExcluding("<Mouse>/position")
            .WithControlsExcluding("<Mouse>/delta")
            .WithControlsExcluding("<Keyboard>/escape")
            .WithCancelingThrough("<Keyboard>/escape")
            .OnMatchWaitForAnother(0.1f)
            .OnComplete(operation =>
            {
                operation.Dispose();
                action.Enable();
                onComplete?.Invoke();
            })
            .OnCancel(operation =>
            {
                operation.Dispose();
                action.Enable();
            })
            .Start();
    }

    public void CancelRebind()
    {
        _rebindOperation?.Cancel();
    }
}
```

### 특정 컨트롤 타입으로 제한

```csharp
// 버튼 컨트롤만 허용
action.PerformInteractiveRebinding()
    .WithExpectedControlType<ButtonControl>()
    .Start();

// 특정 디바이스 제외
action.PerformInteractiveRebinding()
    .WithControlsExcluding("<Mouse>")
    .WithControlsExcluding("<Gamepad>")
    .Start();
```

### Composite Binding 리바인딩

Composite의 개별 파트를 리바인딩할 때는 정확한 바인딩 인덱스를 지정한다.

```csharp
// Move Action의 바인딩 구조 예시:
// [0] 2DVector (composite)
// [1]   Up    (part of composite)
// [2]   Down  (part of composite)
// [3]   Left  (part of composite)
// [4]   Right (part of composite)

// "Up" 키만 리바인딩
action.PerformInteractiveRebinding(1).Start();
```

---

## 바인딩 저장/복원

### JSON으로 저장

```csharp
public static class InputBindingSaveSystem
{
    private const string SaveKey = "input_bindings";

    public static void SaveBindings(InputActionAsset actions)
    {
        string json = actions.SaveBindingOverridesAsJson();
        PlayerPrefs.SetString(SaveKey, json);
        PlayerPrefs.Save();
    }

    public static void LoadBindings(InputActionAsset actions)
    {
        string json = PlayerPrefs.GetString(SaveKey, string.Empty);

        if (!string.IsNullOrEmpty(json))
        {
            actions.LoadBindingOverridesFromJson(json);
        }
    }

    public static void ResetBindings(InputActionAsset actions)
    {
        actions.RemoveAllBindingOverrides();
        PlayerPrefs.DeleteKey(SaveKey);
    }
}
```

### 개별 Action 리셋

```csharp
// 특정 Action의 모든 바인딩 초기화
action.RemoveAllBindingOverrides();

// 특정 바인딩만 초기화
action.RemoveBindingOverride(bindingIndex);
```

---

## 바인딩 표시 문자열

### 현재 바인딩 텍스트 가져오기

```csharp
// 전체 표시 문자열
string display = action.GetBindingDisplayString();

// 특정 바인딩 인덱스의 표시 문자열
string display = action.GetBindingDisplayString(bindingIndex);

// 컨트롤 스킴별 표시
string display = action.GetBindingDisplayString(
    group: "Keyboard&Mouse");
```

### 바인딩 인덱스 찾기

```csharp
// 컨트롤 스킴 기반으로 바인딩 인덱스 검색
int index = action.GetBindingIndex(
    InputBinding.MaskByGroup("Keyboard&Mouse"));

// Composite 파트 이름으로 검색
int index = action.GetBindingIndex(
    InputBinding.MaskByGroup("Keyboard&Mouse")
    + InputBinding.Separator
    + "Up");
```

---

## 리바인딩 UI 구현 패턴

### 리바인딩 버튼 컴포넌트

```csharp
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.UIElements;

public class RebindButton
{
    private readonly InputAction _action;
    private readonly int _bindingIndex;
    private readonly Label _bindingLabel;
    private readonly Button _rebindButton;

    private InputActionRebindingExtensions.RebindingOperation _operation;

    public RebindButton(InputAction action, int bindingIndex,
        Label bindingLabel, Button rebindButton)
    {
        _action = action;
        _bindingIndex = bindingIndex;
        _bindingLabel = bindingLabel;
        _rebindButton = rebindButton;

        _rebindButton.clicked += StartRebind;
        UpdateDisplay();
    }

    private void UpdateDisplay()
    {
        _bindingLabel.text = _action.GetBindingDisplayString(_bindingIndex);
    }

    private void StartRebind()
    {
        _rebindButton.SetEnabled(false);
        _bindingLabel.text = "키를 입력하세요...";

        _action.Disable();

        _operation = _action.PerformInteractiveRebinding(_bindingIndex)
            .WithControlsExcluding("<Mouse>/position")
            .WithControlsExcluding("<Mouse>/delta")
            .WithCancelingThrough("<Keyboard>/escape")
            .OnComplete(_ =>
            {
                CleanupOperation();
                UpdateDisplay();
            })
            .OnCancel(_ =>
            {
                CleanupOperation();
                UpdateDisplay();
            })
            .Start();
    }

    private void CleanupOperation()
    {
        _operation?.Dispose();
        _operation = null;
        _action.Enable();
        _rebindButton.SetEnabled(true);
    }
}
```

---

## 중복 바인딩 검사

### 리바인딩 시 충돌 확인

```csharp
private bool CheckDuplicateBindings(InputAction action, int bindingIndex)
{
    var binding = action.bindings[bindingIndex];
    var actionMap = action.actionMap;

    foreach (var otherAction in actionMap.actions)
    {
        for (int i = 0; i < otherAction.bindings.Count; i++)
        {
            if (otherAction == action && i == bindingIndex)
                continue;

            if (otherAction.bindings[i].effectivePath == binding.effectivePath)
            {
                return true; // 중복 발견
            }
        }
    }

    return false;
}
```
