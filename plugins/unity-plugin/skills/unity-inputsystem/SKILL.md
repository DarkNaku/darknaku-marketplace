---
name: unity-inputsystem
description: >
  Input System 패키지를 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 Action 설계, 바인딩 패턴, 콜백 처리 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - InputAction, InputActionAsset, InputActionMap 언급 시
  - PlayerInput 컴포넌트 사용 또는 설정 작업
  - 키 바인딩, 리바인딩, 컨트롤 스킴 관련 작업
  - 게임패드, 키보드, 마우스, 터치 입력 처리 작업
  - Input System 기반 이동, 공격, UI 입력 구현

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - Input System을 사용하지 않는 프로젝트 (레거시 Input Manager만 사용)
  - 입력과 무관한 순수 게임플레이 로직
user-invocable: false
---

# Unity Input System Skill

## 핵심 원칙

> **Action 기반으로 입력을 추상화한다.**
> 디바이스 컨트롤을 직접 읽지 않고, Action을 통해 입력의 "의도"를 정의한다.
> 이렇게 하면 게임패드, 키보드, 터치 등 여러 디바이스를 하나의 코드로 처리할 수 있다.

> **`FindAction`은 `Start`에서 캐싱한다.**
> `Update`에서 매 프레임 호출하면 성능이 저하된다. 반드시 참조를 멤버 변수에 저장한다.

> **Action은 사용 전에 Enable, 사용 후에 Disable한다.**
> 프로젝트 전역 Action은 기본 활성화되지만, 직접 생성한 Action이나 InputActionAsset은 명시적으로 Enable/Disable 해야 한다.

공식 문서: https://docs.unity3d.com/Packages/com.unity.inputsystem@1.19/manual/index.html

---

## Action 설계

### Action 타입

| 타입 | 용도 | 특징 |
|---|---|---|
| `Value` | 연속 입력 (이동, 시점) | 기본값. 상태 변화 시 `performed` 호출. 초기 상태 체크 수행 |
| `Button` | 일회성 입력 (점프, 공격) | 버튼 컨트롤 전용. 초기 상태 체크 생략 |
| `Pass-Through` | 모든 바운드 컨트롤 | 충돌 해결 없이 모든 입력 전달. 멀티플레이어에 유용 |

### Action Map 구성

용도별로 Action Map을 분리한다.

```
Player          ← 게임플레이 중 입력
├── Move        (Value, Vector2)
├── Look        (Value, Vector2)
├── Jump        (Button)
├── Attack      (Button)
└── Interact    (Button)

UI              ← UI 조작 입력
├── Navigate    (Value, Vector2)
├── Submit      (Button)
├── Cancel      (Button)
└── Point       (Value, Vector2)
```

**원칙:**
- 게임플레이와 UI 입력은 별도 Action Map으로 분리한다
- 상태 전환 시 한 Map을 Disable하고 다른 Map을 Enable한다
- Action 이름은 디바이스가 아닌 의도를 나타낸다 (`PressA` ✗ → `Jump` ✓)

---

## 프로젝트 전역 Action (권장 워크플로우)

### 기본 사용법

Edit > Project Settings > Input System Package > Input Actions에서 설정한 프로젝트 전역 Action을 사용한다.

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    private InputAction _moveAction;
    private InputAction _jumpAction;

    private void Start()
    {
        _moveAction = InputSystem.actions.FindAction("Move");
        _jumpAction = InputSystem.actions.FindAction("Jump");
    }

    private void Update()
    {
        Vector2 moveValue = _moveAction.ReadValue<Vector2>();
        transform.Translate(new Vector3(moveValue.x, 0, moveValue.y) * Time.deltaTime);

        if (_jumpAction.WasPressedThisFrame())
        {
            Jump();
        }
    }
}
```

---

## InputActionAsset 워크플로우

### C# 클래스 생성

`.inputactions` 에셋의 Inspector에서 **Generate C# Class**를 활성화하면 타입 안전한 래퍼 클래스가 생성된다.

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour, MyControls.IPlayerActions
{
    private MyControls _controls;

    private void OnEnable()
    {
        if (_controls == null)
        {
            _controls = new MyControls();
            _controls.Player.SetCallbacks(this);
        }
        _controls.Player.Enable();
    }

    private void OnDisable()
    {
        _controls.Player.Disable();
    }

    public void OnMove(InputAction.CallbackContext context)
    {
        Vector2 moveValue = context.ReadValue<Vector2>();
    }

    public void OnJump(InputAction.CallbackContext context)
    {
        if (context.performed)
        {
            Jump();
        }
    }
}
```

**장점:**
- 문자열 기반 `FindAction` 제거 (타입 안전)
- 인터페이스로 콜백 강제 (누락 방지)
- Action Map별 Enable/Disable 가능

---

## 입력 응답 패턴

### 폴링 방식 (연속 입력에 적합)

```csharp
private void Update()
{
    // 연속 값 읽기 (이동, 시점)
    Vector2 move = _moveAction.ReadValue<Vector2>();

    // 이번 프레임에 수행되었는지
    if (_jumpAction.WasPerformedThisFrame()) { }

    // 버튼 눌림/해제 감지
    if (_attackAction.WasPressedThisFrame()) { }
    if (_attackAction.WasReleasedThisFrame()) { }

    // 현재 눌려 있는지
    if (_sprintAction.IsPressed()) { }
}
```

### 콜백 방식 (이벤트 기반 입력에 적합)

```csharp
private void OnEnable()
{
    _jumpAction.performed += OnJumpPerformed;
    _jumpAction.canceled += OnJumpCanceled;
}

private void OnDisable()
{
    _jumpAction.performed -= OnJumpPerformed;
    _jumpAction.canceled -= OnJumpCanceled;
}

private void OnJumpPerformed(InputAction.CallbackContext context)
{
    // 점프 실행
}

private void OnJumpCanceled(InputAction.CallbackContext context)
{
    // 점프 취소 처리
}
```

### Action Phase

| Phase | 의미 | 용도 |
|---|---|---|
| `Disabled` | 비활성 상태 | 입력을 받지 않음 |
| `Waiting` | 대기 중 | 활성화됨, 입력 감시 중 |
| `Started` | 입력 시작 | Interaction 시작됨 |
| `Performed` | 수행 완료 | Interaction 완료 (주요 응답 지점) |
| `Canceled` | 취소됨 | Interaction 중단됨 |

---

## Composite Binding

### 2D Vector (WASD 이동)

```csharp
var moveAction = new InputAction("Move", InputActionType.Value);
moveAction.AddCompositeBinding("2DVector")
    .With("Up", "<Keyboard>/w")
    .With("Down", "<Keyboard>/s")
    .With("Left", "<Keyboard>/a")
    .With("Right", "<Keyboard>/d");
```

### 1D Axis

```csharp
var zoomAction = new InputAction("Zoom");
zoomAction.AddCompositeBinding("Axis")
    .With("Positive", "<Keyboard>/equals")
    .With("Negative", "<Keyboard>/minus");
```

### Modifier (Shift + 키)

```csharp
var sprintAction = new InputAction("Sprint");
sprintAction.AddCompositeBinding("OneModifier")
    .With("Modifier", "<Keyboard>/leftShift")
    .With("Binding", "<Keyboard>/w");
```

---

## Interaction

### 내장 Interaction 타입

| 타입 | 동작 | 용도 |
|---|---|---|
| `Press` | 버튼 누름/떼기 감지 | 정밀한 Press/Release 제어 |
| `Hold` | 일정 시간 누르고 있기 | 차징, 장기 입력 |
| `Tap` | 빠르게 눌렀다 떼기 | 탭 제스처 |
| `SlowTap` | 길게 눌렀다 떼기 | 길게 누르기 제스처 |
| `MultiTap` | 연속 탭 | 더블 클릭, 트리플 탭 |

### 코드에서 Interaction 설정

```csharp
// Hold: 0.5초 이상 누르면 performed
action.AddBinding("<Gamepad>/buttonSouth")
    .WithInteractions("hold(duration=0.5)");

// Tap + SlowTap 구분
action.AddBinding("<Gamepad>/buttonSouth")
    .WithInteractions("tap,slowTap(duration=0.5)");
```

### Interaction 콜백 활용

```csharp
// 차징 UI 피드백
action.started += _ => ShowChargeUI();
action.performed += _ => ExecuteChargedAttack();
action.canceled += _ => HideChargeUI();

// 차징 진행률 확인
float progress = action.GetTimeoutCompletionPercentage();
```

---

## Action Map 전환

### 상태별 Map 전환 패턴

```csharp
public class InputMapSwitcher
{
    private readonly InputActionAsset _actions;

    public InputMapSwitcher(InputActionAsset actions)
    {
        _actions = actions;
    }

    public void SwitchToGameplay()
    {
        _actions.FindActionMap("UI").Disable();
        _actions.FindActionMap("Player").Enable();
    }

    public void SwitchToUI()
    {
        _actions.FindActionMap("Player").Disable();
        _actions.FindActionMap("UI").Enable();
    }
}
```

### 생성된 C# 클래스 사용 시

```csharp
public void SwitchToGameplay()
{
    _controls.UI.Disable();
    _controls.Player.Enable();
}
```

---

## PlayerInput 컴포넌트

### Behavior 옵션

| 옵션 | 방식 | 적합한 상황 |
|---|---|---|
| `Send Messages` | `SendMessage`로 호출 | 단순 프로토타이핑 |
| `Broadcast Messages` | `BroadcastMessage`로 호출 | 자식 오브젝트에도 전달 필요 시 |
| `Invoke Unity Events` | UnityEvent 연결 | Inspector에서 시각적 연결 |
| `Invoke C Sharp Events` | C# 이벤트 | 코드 기반 구독 (권장) |

### Send Messages 방식

```csharp
public void OnMove(InputValue value)
{
    Vector2 moveValue = value.Get<Vector2>();
}

public void OnJump()
{
    Jump();
}
```

### Invoke Unity Events 방식

```csharp
public void OnMove(InputAction.CallbackContext context)
{
    Vector2 moveValue = context.ReadValue<Vector2>();
}
```

### 주의사항

- PlayerInput은 Action 사본을 생성한다. `InputSystem.actions`가 아닌 `playerInput.actions`로 접근해야 한다
- 멀티플레이어에서는 PlayerInput + PlayerInputManager를 사용한다
- 단일 플레이어 게임에서는 프로젝트 전역 Action 또는 InputActionAsset이 더 간결하다

---

## 터치 입력

### EnhancedTouch API

```csharp
using UnityEngine.InputSystem.EnhancedTouch;
using Touch = UnityEngine.InputSystem.EnhancedTouch.Touch;

private void OnEnable()
{
    EnhancedTouchSupport.Enable();
}

private void OnDisable()
{
    EnhancedTouchSupport.Disable();
}

private void Update()
{
    foreach (var touch in Touch.activeTouches)
    {
        switch (touch.phase)
        {
            case UnityEngine.InputSystem.TouchPhase.Began:
                OnTouchStart(touch.screenPosition);
                break;
            case UnityEngine.InputSystem.TouchPhase.Moved:
                OnTouchMove(touch.screenPosition, touch.delta);
                break;
            case UnityEngine.InputSystem.TouchPhase.Ended:
                OnTouchEnd(touch.screenPosition);
                break;
        }
    }
}
```

### 터치 시뮬레이션 (에디터 테스트)

```csharp
using UnityEngine.InputSystem;

// 마우스 입력을 터치로 시뮬레이션
TouchSimulation.Enable();
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `Update`에서 `FindAction` 호출 | `Start`에서 캐싱하여 멤버 변수에 저장 |
| 디바이스 컨트롤 직접 읽기 (`Gamepad.current.leftStick`) | Action을 통해 추상화 |
| Action Enable 없이 입력 읽기 | 사용 전 `action.Enable()` 또는 `actionMap.Enable()` |
| 콜백 등록 후 해제 누락 | `OnDisable`에서 반드시 `-=`로 구독 해제 |
| Action 활성 상태에서 바인딩 수정 | 바인딩 수정 전 Disable, 수정 후 Enable |
| `InputSystem.actions`로 PlayerInput의 Action 접근 | `playerInput.actions` 사용 |
| Action 이름에 디바이스명 사용 (`PressA`, `ClickMouse`) | 의도를 나타내는 이름 (`Jump`, `Select`) |
| `async void`에서 입력 처리 | 동기 콜백으로 처리 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| 런타임 리바인딩 UI 구현, 바인딩 저장/복원, 표시 문자열 처리 | `references/rebinding.md` |
| VContainer 연동, 입력 서비스 추상화, EntryPoint에서 입력 처리 | `references/di-integration.md` |
