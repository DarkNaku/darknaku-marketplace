# VContainer 연동 패턴

## 입력 서비스 추상화

입력 처리를 인터페이스로 추상화하여 DI 컨테이너에 등록한다.

### 인터페이스 정의

```csharp
using System;
using UnityEngine;
using UnityEngine.InputSystem;

public interface IInputService
{
    Vector2 MoveInput { get; }
    Vector2 LookInput { get; }
    event Action OnJump;
    event Action OnAttack;
    event Action<bool> OnSprint;
    void EnableGameplay();
    void DisableGameplay();
}
```

### 구현 (프로젝트 전역 Action)

```csharp
using System;
using UnityEngine;
using UnityEngine.InputSystem;

public class InputService : IInputService, IDisposable
{
    private readonly InputAction _moveAction;
    private readonly InputAction _lookAction;
    private readonly InputAction _jumpAction;
    private readonly InputAction _attackAction;
    private readonly InputAction _sprintAction;

    public Vector2 MoveInput => _moveAction.ReadValue<Vector2>();
    public Vector2 LookInput => _lookAction.ReadValue<Vector2>();

    public event Action OnJump;
    public event Action OnAttack;
    public event Action<bool> OnSprint;

    public InputService()
    {
        _moveAction = InputSystem.actions.FindAction("Move");
        _lookAction = InputSystem.actions.FindAction("Look");
        _jumpAction = InputSystem.actions.FindAction("Jump");
        _attackAction = InputSystem.actions.FindAction("Attack");
        _sprintAction = InputSystem.actions.FindAction("Sprint");

        _jumpAction.performed += HandleJump;
        _attackAction.performed += HandleAttack;
        _sprintAction.performed += HandleSprintStart;
        _sprintAction.canceled += HandleSprintEnd;
    }

    public void EnableGameplay()
    {
        _moveAction.Enable();
        _lookAction.Enable();
        _jumpAction.Enable();
        _attackAction.Enable();
        _sprintAction.Enable();
    }

    public void DisableGameplay()
    {
        _moveAction.Disable();
        _lookAction.Disable();
        _jumpAction.Disable();
        _attackAction.Disable();
        _sprintAction.Disable();
    }

    private void HandleJump(InputAction.CallbackContext context) => OnJump?.Invoke();
    private void HandleAttack(InputAction.CallbackContext context) => OnAttack?.Invoke();
    private void HandleSprintStart(InputAction.CallbackContext context) => OnSprint?.Invoke(true);
    private void HandleSprintEnd(InputAction.CallbackContext context) => OnSprint?.Invoke(false);

    public void Dispose()
    {
        _jumpAction.performed -= HandleJump;
        _attackAction.performed -= HandleAttack;
        _sprintAction.performed -= HandleSprintStart;
        _sprintAction.canceled -= HandleSprintEnd;
    }
}
```

### 구현 (InputActionAsset)

```csharp
using System;
using UnityEngine;
using UnityEngine.InputSystem;

public class AssetInputService : IInputService, IDisposable
{
    private readonly InputActionAsset _actions;
    private readonly InputActionMap _playerMap;
    private readonly InputAction _moveAction;
    private readonly InputAction _lookAction;
    private readonly InputAction _jumpAction;
    private readonly InputAction _attackAction;
    private readonly InputAction _sprintAction;

    public Vector2 MoveInput => _moveAction.ReadValue<Vector2>();
    public Vector2 LookInput => _lookAction.ReadValue<Vector2>();

    public event Action OnJump;
    public event Action OnAttack;
    public event Action<bool> OnSprint;

    public AssetInputService(InputActionAsset actions)
    {
        _actions = actions;
        _playerMap = _actions.FindActionMap("Player");
        _moveAction = _playerMap.FindAction("Move");
        _lookAction = _playerMap.FindAction("Look");
        _jumpAction = _playerMap.FindAction("Jump");
        _attackAction = _playerMap.FindAction("Attack");
        _sprintAction = _playerMap.FindAction("Sprint");

        _jumpAction.performed += HandleJump;
        _attackAction.performed += HandleAttack;
        _sprintAction.performed += HandleSprintStart;
        _sprintAction.canceled += HandleSprintEnd;
    }

    public void EnableGameplay() => _playerMap.Enable();
    public void DisableGameplay() => _playerMap.Disable();

    private void HandleJump(InputAction.CallbackContext context) => OnJump?.Invoke();
    private void HandleAttack(InputAction.CallbackContext context) => OnAttack?.Invoke();
    private void HandleSprintStart(InputAction.CallbackContext context) => OnSprint?.Invoke(true);
    private void HandleSprintEnd(InputAction.CallbackContext context) => OnSprint?.Invoke(false);

    public void Dispose()
    {
        _playerMap.Disable();
        _jumpAction.performed -= HandleJump;
        _attackAction.performed -= HandleAttack;
        _sprintAction.performed -= HandleSprintStart;
        _sprintAction.canceled -= HandleSprintEnd;
    }
}
```

---

## LifetimeScope 등록

### 프로젝트 전역 Action 방식

```csharp
using VContainer;
using VContainer.Unity;

public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<IInputService, InputService>(Lifetime.Singleton);
    }
}
```

### InputActionAsset 방식

```csharp
using UnityEngine;
using VContainer;
using VContainer.Unity;
using UnityEngine.InputSystem;

public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private InputActionAsset _inputActions;

    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterInstance(_inputActions);
        builder.Register<IInputService, AssetInputService>(Lifetime.Singleton);
    }
}
```

---

## EntryPoint에서 입력 처리

### Presenter 패턴

```csharp
using VContainer.Unity;

public class PlayerPresenter : IStartable, ITickable, IDisposable
{
    private readonly IInputService _input;
    private readonly PlayerModel _model;
    private readonly PlayerView _view;

    public PlayerPresenter(IInputService input, PlayerModel model, PlayerView view)
    {
        _input = input;
        _model = model;
        _view = view;
    }

    void IStartable.Start()
    {
        _input.OnJump += HandleJump;
        _input.OnAttack += HandleAttack;
        _input.OnSprint += HandleSprint;
        _input.EnableGameplay();
    }

    void ITickable.Tick()
    {
        _model.Move(_input.MoveInput);
        _model.Look(_input.LookInput);
        _view.UpdatePosition(_model.Position);
    }

    private void HandleJump() => _model.Jump();
    private void HandleAttack() => _model.Attack();
    private void HandleSprint(bool active) => _model.SetSprint(active);

    void IDisposable.Dispose()
    {
        _input.OnJump -= HandleJump;
        _input.OnAttack -= HandleAttack;
        _input.OnSprint -= HandleSprint;
        _input.DisableGameplay();
    }
}
```

---

## Action Map 전환 서비스

### 인터페이스

```csharp
public interface IInputMapService
{
    void SwitchToGameplay();
    void SwitchToUI();
    void DisableAll();
}
```

### 구현

```csharp
using System;
using UnityEngine.InputSystem;

public class InputMapService : IInputMapService
{
    private readonly InputActionAsset _actions;
    private readonly InputActionMap _playerMap;
    private readonly InputActionMap _uiMap;

    public InputMapService(InputActionAsset actions)
    {
        _actions = actions;
        _playerMap = _actions.FindActionMap("Player");
        _uiMap = _actions.FindActionMap("UI");
    }

    public void SwitchToGameplay()
    {
        _uiMap.Disable();
        _playerMap.Enable();
    }

    public void SwitchToUI()
    {
        _playerMap.Disable();
        _uiMap.Enable();
    }

    public void DisableAll()
    {
        _playerMap.Disable();
        _uiMap.Disable();
    }
}
```

### 등록

```csharp
builder.Register<IInputMapService, InputMapService>(Lifetime.Singleton);
```
