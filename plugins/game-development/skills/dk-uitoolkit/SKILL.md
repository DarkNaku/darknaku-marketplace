---
name: dk-uitoolkit
description: >
  Unity 프로젝트에서 UI를 만들거나 수정할 때 반드시 사용하는 스킬.
  uGUI(Canvas/GameObject 기반 UI)가 아닌 UI Toolkit(UXML + USS + C#)을 사용하도록 강제하고,
  올바른 파일 구조, 컴포넌트 패턴, VContainer/R3/MessagePipe/MVP 연동 방식을 안내한다.
  
  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - "UI 만들어줘", "버튼/슬라이더/팝업 추가", "HUD 구현", "인벤토리 화면"
  - "인스펙터 커스터마이징", "EditorWindow 만들기"
  - Canvas, Image, Text(TMP) 같은 uGUI 컴포넌트 언급 시 → UI Toolkit으로 유도
  - UXML, USS, VisualElement 관련 작업
  - 기존 UI 코드 수정/리팩터링
---

# Unity UI Toolkit Skill

## 핵심 원칙

> **uGUI(Canvas 기반)는 사용하지 않는다.**  
> 모든 UI는 UI Toolkit(UXML + USS + C#)으로 구현한다.  
> 단, TextMeshPro 월드 스페이스 텍스트나 파티클 위 오버레이처럼 UI Toolkit이 기술적으로 불가능한 케이스에 한해 예외를 허용하며, 이 경우 반드시 사용자에게 이유를 설명한다.

---

## 파일 구조

```
Assets/
└── UI/
    ├── Layouts/           # UXML 파일 (구조 정의)
    │   └── HudView.uxml
    ├── Styles/            # USS 파일 (스타일 정의)
    │   ├── Variables.uss  # 색상·폰트 등 변수 모음
    │   └── HudView.uss
    ├── Themes/            # 테마 파일 (선택)
    └── Scripts/
        ├── Views/         # VisualElement 래퍼 (View)
        │   └── HudView.cs
        └── Presenters/    # 비즈니스 로직 연결 (Presenter)
            └── HudPresenter.cs
```

---

## UXML 작성 규칙

```xml
<!-- HudView.uxml -->
<ui:UXML xmlns:ui="UnityEngine.UIElements"
         xmlns:uie="UnityEditor.UIElements"
         xsi:noNamespaceSchemaLocation="../../../UIElementsSchema/UIElements.xsd">

    <ui:VisualElement name="hud-root" class="hud-root">
        <ui:Label name="score-label" class="score-label" text="0" />
        <ui:VisualElement name="health-bar-container" class="health-bar">
            <ui:VisualElement name="health-bar-fill" class="health-bar__fill" />
        </ui:VisualElement>
        <ui:Button name="pause-button" class="btn btn--icon" text="II" />
    </ui:VisualElement>

</ui:UXML>
```

**규칙:**
- `name` 속성: C#에서 `Q<T>("name")`으로 참조 → **camelCase 금지, kebab-case 사용**
- `class` 속성: USS 스타일링용 → BEM 방식 권장 (`block__element--modifier`)
- 텍스트 콘텐츠는 `text=""` 초기값만 설정, 실제 값은 C#에서 바인딩

---

## USS 작성 규칙

```css
/* Variables.uss */
:root {
    --color-primary: #4A90E2;
    --color-danger: #E24A4A;
    --font-size-base: 16px;
    --spacing-sm: 8px;
    --spacing-md: 16px;
}

/* HudView.uss */
.hud-root {
    flex-direction: row;
    align-items: center;
    padding: var(--spacing-sm);
    background-color: rgba(0, 0, 0, 0.5);
}

.score-label {
    color: white;
    font-size: var(--font-size-base);
    -unity-font-style: bold;
    min-width: 80px;
}

.health-bar {
    flex: 1;
    height: 12px;
    background-color: rgba(255, 255, 255, 0.2);
    border-radius: 6px;
    overflow: hidden;
}

.health-bar__fill {
    height: 100%;
    background-color: var(--color-primary);
    transition: width 0.3s ease;
}

.btn {
    background-color: var(--color-primary);
    color: white;
    border-width: 0;
    border-radius: 4px;
    padding: var(--spacing-sm) var(--spacing-md);
    cursor: pointer;
}

.btn:hover {
    background-color: rgba(74, 144, 226, 0.8);
}

.btn:active {
    scale: 0.95;
}
```

**USS 주의사항:**
- `width`, `height`는 px 또는 % 단위 명시 필수
- Flexbox가 기본 레이아웃 시스템 (Unity는 Yoga 엔진)
- `position: absolute`는 HUD 오버레이에만 제한적으로 사용
- 애니메이션은 `transition` 속성 사용 (USS Transitions)
- `-unity-` 접두사 속성: Unity 전용 확장 (예: `-unity-font-style`, `-unity-text-align`)

---

## C# View 패턴 (VisualElement 래퍼)

```csharp
// HudView.cs
using UnityEngine;
using UnityEngine.UIElements;

public class HudView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;

    // UI 요소 캐싱
    private Label _scoreLabel;
    private VisualElement _healthBarFill;
    private Button _pauseButton;

    // 이벤트 노출 (Presenter가 구독)
    public event System.Action OnPauseClicked;

    private void Awake()
    {
        var root = _uiDocument.rootVisualElement;

        // Q<T>() 로 요소 쿼리 — 항상 캐싱할 것
        _scoreLabel     = root.Q<Label>("score-label");
        _healthBarFill  = root.Q<VisualElement>("health-bar-fill");
        _pauseButton    = root.Q<Button>("pause-button");

        // 이벤트 등록
        _pauseButton.clicked += () => OnPauseClicked?.Invoke();
    }

    // 상태 업데이트 메서드 (Presenter가 호출)
    public void SetScore(int score)
        => _scoreLabel.text = score.ToString("N0");

    public void SetHealthRatio(float ratio)
        => _healthBarFill.style.width = Length.Percent(ratio * 100f);

    private void OnDestroy()
    {
        _pauseButton.clicked -= () => OnPauseClicked?.Invoke();
    }
}
```

---

## MVP + VContainer + R3 연동 패턴

```csharp
// HudPresenter.cs
using System;
using VContainer;
using VContainer.Unity;
using R3;

public class HudPresenter : IStartable, IDisposable
{
    private readonly HudView _view;
    private readonly GameStateModel _gameState;   // 도메인 모델 (Core 레이어)
    private readonly CompositeDisposable _disposables = new();

    [Inject]
    public HudPresenter(HudView view, GameStateModel gameState)
    {
        _view = view;
        _gameState = gameState;
    }

    public void Start()
    {
        // R3로 상태 변화 구독 → View 업데이트
        _gameState.Score
            .Subscribe(score => _view.SetScore(score))
            .AddTo(_disposables);

        _gameState.HealthRatio
            .Subscribe(ratio => _view.SetHealthRatio(ratio))
            .AddTo(_disposables);

        // View 이벤트 → 도메인 커맨드
        _view.OnPauseClicked += HandlePause;
    }

    private void HandlePause()
        => _gameState.RequestPause();

    public void Dispose()
    {
        _view.OnPauseClicked -= HandlePause;
        _disposables.Dispose();
    }
}
```

```csharp
// GameLifetimeScope.cs (VContainer)
public class GameLifetimeScope : LifetimeScope
{
    [SerializeField] private HudView _hudView;

    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterInstance(_hudView);
        builder.Register<GameStateModel>(Lifetime.Singleton);
        builder.RegisterEntryPoint<HudPresenter>();
    }
}
```

---

## 팝업 / 동적 UI 패턴

```csharp
// 팝업을 동적으로 생성할 때
public class PopupService
{
    private readonly VisualElement _overlayRoot;
    private readonly VisualTreeAsset _popupTemplate;  // UXML Asset

    public void ShowPopup(string title, string message)
    {
        // UXML 클론 → 오버레이에 추가
        var popup = _popupTemplate.Instantiate();
        popup.Q<Label>("popup-title").text = title;
        popup.Q<Label>("popup-message").text = message;
        popup.Q<Button>("popup-close").clicked += () => ClosePopup(popup);

        _overlayRoot.Add(popup);

        // 등장 애니메이션 (USS transition 활용)
        popup.schedule.Execute(() =>
            popup.AddToClassList("popup--visible")).StartingIn(0);
    }

    private void ClosePopup(VisualElement popup)
    {
        popup.RemoveFromClassList("popup--visible");
        // transition 완료 후 제거
        popup.schedule.Execute(() => popup.RemoveFromHierarchy())
            .StartingIn(300); // transition duration과 맞출 것
    }
}
```

---

## 에디터 UI (EditorWindow / CustomInspector)

```csharp
// CustomEditorWindow.cs
using UnityEditor;
using UnityEngine.UIElements;
using UnityEditor.UIElements;

public class MyToolWindow : EditorWindow
{
    [MenuItem("Tools/My Tool")]
    public static void ShowWindow()
        => GetWindow<MyToolWindow>("My Tool");

    public override void CreateGUI()
    {
        // UXML 로드 방식 (에디터)
        var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>(
            "Assets/UI/Editor/MyToolWindow.uxml");
        rootVisualElement.Add(visualTree.Instantiate());

        // USS 로드
        var styleSheet = AssetDatabase.LoadAssetAtPath<StyleSheet>(
            "Assets/UI/Editor/MyToolWindow.uss");
        rootVisualElement.styleSheets.Add(styleSheet);

        // 요소 쿼리 및 이벤트
        rootVisualElement.Q<Button>("run-button").clicked += OnRunClicked;
    }

    private void OnRunClicked() { /* ... */ }
}
```

```csharp
// CustomInspector.cs
using UnityEditor;
using UnityEngine.UIElements;

[CustomEditor(typeof(MyComponent))]
public class MyComponentEditor : Editor
{
    public override VisualElement CreateInspectorGUI()
    {
        var root = new VisualElement();

        // 기본 인스펙터 유지하려면
        InspectorElement.FillDefaultInspector(root, serializedObject, this);

        // 커스텀 버튼 추가
        var btn = new Button(() => Debug.Log("Custom action"))
        {
            text = "Do Something"
        };
        root.Add(btn);

        return root;
    }
}
```

---

## 공식 예제 저장소 참조

복잡한 컴포넌트 구현 시 반드시 공식 예제를 참고한다:  
📦 `https://github.com/Unity-Technologies/ui-toolkit-manual-code-examples`

주요 폴더 구조:
| 폴더 | 내용 |
|---|---|
| `bind-custom-control/` | 커스텀 컨트롤 데이터 바인딩 |
| `ListViewExample/` | ListView 아이템 바인딩 패턴 |
| `ToggleExample/` | Toggle 커스텀 구현 |
| `create-custom-controls/` | VisualElement 상속 커스텀 컨트롤 |

→ 구체적인 예제 코드가 필요할 때는 해당 저장소의 raw URL을 `web_fetch`로 가져와서 참고한다.

---

## 자주 하는 실수 / 금지 패턴

| ❌ 하지 말 것 | ✅ 대신 할 것 |
|---|---|
| `GameObject.Find()` + `GetComponent<Image>()` | `root.Q<VisualElement>("name")` |
| `Update()`에서 매 프레임 `Q<>()` 호출 | `Awake()`에서 한 번만 캐싱 |
| USS 없이 C#에서 인라인 스타일 남발 | USS 클래스로 스타일 분리 |
| `Canvas/RectTransform` 사용 | `UIDocument` + UXML |
| `Text` 또는 `TMP_Text` 컴포넌트 | `Label` VisualElement |
| 이벤트 등록 후 `OnDestroy`에서 해제 안 함 | 반드시 `clicked -=` 또는 Disposable 패턴 |
| 팝업을 Instantiate(prefab) 으로 생성 | `VisualTreeAsset.Instantiate()` |

---

## 상세 참조

더 복잡한 케이스는 references/ 폴더를 확인한다:
- `references/data-binding.md` — Unity 6+ 공식 데이터 바인딩 API
- `references/custom-controls.md` — VisualElement 상속 커스텀 컨트롤 패턴
- `references/performance.md` — UI Toolkit 성능 최적화 팁
