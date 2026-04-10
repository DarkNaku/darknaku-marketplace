---
name: dk-uitoolkit
description: >
  UI Toolkit(UXML + USS + C#)을 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 파일 구조, 컴포넌트 패턴, MVP 아키텍처 연동 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - UXML, USS, VisualElement, UIDocument, UIToolkit 언급 시
  - VisualElement 상속, Q<T>() 쿼리, USS 클래스 토글 관련 작업
  - UI Toolkit 기반 View/Presenter 구현 및 리팩터링
  
  다음 상황에서는 이 스킬을 참조하지 않는다:
  - Canvas, Image, RectTransform 등 uGUI 작업 (별도 안내)
  - UI와 무관한 게임플레이 로직
user-invocable: false
---

# Unity UI Toolkit Skill

## 핵심 원칙

> **uGUI(Canvas 기반)는 사용하지 않는다.**  
> 모든 UI는 UI Toolkit(UXML + USS + C#)으로 구현한다.  
> 단, TextMeshPro 월드 스페이스 텍스트나 파티클 위 오버레이처럼 UI Toolkit이 기술적으로 불가능한 케이스에 한해 예외를 허용하며, 이 경우 반드시 사용자에게 이유를 설명한다.

---

## UI 구현 전 체크리스트

코드 작성 전에 다음을 확인한다. 이 단계를 건너뛰면 구현 도중 방향이 바뀌거나 재작업이 발생한다.

- **플레이어 컨텍스트**: 이 UI가 표시될 때 플레이어 상태는? (전투 중 / 일시정지 / 메뉴 / 컷씬)  
  → 전투 중이라면 레이아웃 shift나 시선을 빼앗는 애니메이션은 피한다.
- **시선 우선순위**: 가장 먼저 읽혀야 할 정보는 무엇인가?  
  → Visual Hierarchy를 USS `font-size`, `color`, `z-order`로 반영한다.
- **입력 방식**: 마우스+키보드 / 게임패드 / 터치 중 무엇이 주인가?  
  → USS의 `:hover`는 게임패드에서 동작하지 않는다. 포커스 스타일(`:focus`)을 별도로 작성해야 한다.
- **해상도 대응**: px 고정값인가, flex+%인가?  
  → Panel Settings의 `Scale Mode`를 먼저 결정한다 (`Scale With Screen Size` 권장).
- **공용 스타일 확인**: `Variables.uss`의 토큰을 먼저 확인한다.  
  → 새로운 색상·간격 값이 필요하면 하드코딩 전에 토큰으로 추가한다.

---

## 파일 구조

```
Assets/
└── UI/
    ├── Layouts/           # UXML 파일 — 구조(HTML 역할)
    │   └── HudView.uxml
    ├── Styles/            # USS 파일 — 스타일(CSS 역할)
    │   ├── Variables.uss  # 디자인 토큰 (색상·폰트·간격 변수)
    │   └── HudView.uss
    ├── Themes/            # 테마 파일 (다크/라이트 등, 선택)
    └── Scripts/
        ├── Views/         # VisualElement 래퍼 — UI 상태 표현만 담당
        │   └── HudView.cs
        └── Presenters/    # 비즈니스 로직 연결 — View ↔ Model 중재
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

### Variables.uss — 디자인 토큰 체계

토큰은 두 레이어로 관리한다. 새로운 색상이 필요할 때 하드코딩하지 않고 여기에 먼저 추가한다.

```css
/* Variables.uss */
:root {
    /* ── Primitive tokens: 팔레트 원색, 의미 없이 값 그 자체 ── */
    --palette-blue-500: #4A90E2;
    --palette-blue-700: #2E6DB4;
    --palette-red-500:  #E24A4A;
    --palette-gray-900: rgba(0, 0, 0, 0.5);
    --palette-gray-200: rgba(255, 255, 255, 0.2);

    /* ── Semantic tokens: 역할 기반 참조 ── */
    /* 색상 */
    --color-action-primary:  var(--palette-blue-500);  /* 주요 버튼, 강조 */
    --color-action-hover:    var(--palette-blue-700);  /* 버튼 호버 */
    --color-feedback-danger: var(--palette-red-500);   /* HP 위험, 에러 */
    --color-surface-hud:     var(--palette-gray-900);  /* HUD 배경 */
    --color-surface-overlay: var(--palette-gray-200);  /* 바 배경 등 */

    /* 폰트 */
    --font-size-sm:   12px;
    --font-size-base: 16px;
    --font-size-lg:   24px;

    /* 간격 (4px 기반 스케일) */
    --space-1: 4px;
    --space-2: 8px;
    --space-4: 16px;
    --space-8: 32px;

    /* 모션 */
    --duration-fast:   150ms;  /* HUD 즉각 반응 */
    --duration-normal: 250ms;  /* 팝업 등장 */
    --duration-slow:   350ms;  /* 팝업 퇴장 */
    --easing-default:  ease-out;
}
```

> **Primitive vs Semantic**: 버튼 색을 파란색 → 초록색으로 바꿀 때, Primitive를 직접 쓰면 파일 전체를 검색해야 한다. Semantic 토큰(`--color-action-primary`)을 쓰면 Variables.uss 한 줄만 수정하면 된다.

### 컴포넌트 스타일 (HudView.uss)

```css
/* HudView.uss */
.hud-root {
    flex-direction: row;
    align-items: center;
    padding: var(--space-2);
    background-color: var(--color-surface-hud);
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
    background-color: var(--color-surface-overlay);
    border-radius: 6px;
    overflow: hidden;
}

.health-bar__fill {
    height: 100%;
    background-color: var(--color-action-primary);
    transition: width var(--duration-fast) var(--easing-default);
}

.btn {
    background-color: var(--color-action-primary);
    color: white;
    border-width: 0;
    border-radius: 4px;
    padding: var(--space-2) var(--space-4);
    cursor: pointer;
    transition: background-color var(--duration-fast) var(--easing-default),
                scale var(--duration-fast) var(--easing-default);
}

.btn:hover  { background-color: var(--color-action-hover); }
.btn:focus  { background-color: var(--color-action-hover); } /* 게임패드 대응 */
.btn:active { scale: 0.95; }
```

**USS 주의사항:**
- `width`, `height`는 px 또는 % 단위 명시 필수
- Flexbox가 기본 레이아웃 시스템 (Unity는 Yoga 엔진)
- `position: absolute`는 HUD 오버레이에만 제한적으로 사용
- `-unity-` 접두사 속성: Unity 전용 확장 (예: `-unity-font-style`, `-unity-text-align`)

### Motion 원칙

| 컨텍스트 | 권장 duration | 이유 |
|---|---|---|
| HUD 수치 변화 (HP, 점수) | `--duration-fast` (150ms) | 플레이 반응성 우선 |
| 팝업 등장 | `--duration-normal` (250ms) | 존재감 있되 방해되지 않게 |
| 팝업 퇴장 | `--duration-slow` (350ms) | 사라지는 느낌을 자연스럽게 |
| 전투 중 UI 전환 | 최소화 또는 없앰 | 시선 분산 방지 |

USS `transition`으로 해결 가능한 애니메이션은 C# `schedule`을 쓰지 않는다. 복잡한 시퀀스만 `schedule`로 처리한다. 자세한 비교는 `references/performance.md` 참조.

---

## C# View 패턴 (VisualElement 래퍼)

```csharp
// HudView.cs
using UnityEngine;
using UnityEngine.UIElements;

public class HudView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;

    // UI 요소 캐싱 — Q<>()는 Awake에서 한 번만 호출
    private Label _scoreLabel;
    private VisualElement _healthBarFill;
    private Button _pauseButton;

    // 이벤트 노출 (Presenter가 구독)
    public event System.Action OnPauseClicked;

    private void Awake()
    {
        var root = _uiDocument.rootVisualElement;

        _scoreLabel    = root.Q<Label>("score-label");
        _healthBarFill = root.Q<VisualElement>("health-bar-fill");
        _pauseButton   = root.Q<Button>("pause-button");

        // 람다 대신 메서드 참조 — OnDestroy에서 정확히 해제하기 위해 필수
        _pauseButton.clicked += OnPauseButtonClicked;
    }

    public void SetScore(int score)
        => _scoreLabel.text = score.ToString("N0");

    public void SetHealthRatio(float ratio)
        => _healthBarFill.style.width = Length.Percent(ratio * 100f);

    private void OnPauseButtonClicked()
        => OnPauseClicked?.Invoke();

    private void OnDestroy()
    {
        // 람다로 -= 하면 실제로 해제되지 않는다 (새 인스턴스이므로)
        // 반드시 등록한 것과 같은 메서드 참조로 해제할 것
        _pauseButton.clicked -= OnPauseButtonClicked;
    }
}
```

---

## MVP 패턴

View는 UI 상태 표현만 담당하고, Presenter가 Model과 View를 중재한다.  
DI 컨테이너나 Rx 라이브러리는 프로젝트 환경에 맞게 선택한다. 아래는 순수 C#으로 작성한 기본 패턴이다.

```csharp
// HudPresenter.cs
using System;

public class HudPresenter : IDisposable
{
    private readonly HudView _view;
    private readonly GameStateModel _model;

    public HudPresenter(HudView view, GameStateModel model)
    {
        _view  = view;
        _model = model;

        // Model 이벤트 → View 업데이트
        _model.OnScoreChanged    += _view.SetScore;
        _model.OnHealthChanged   += _view.SetHealthRatio;

        // View 이벤트 → Model 커맨드
        _view.OnPauseClicked     += HandlePause;
    }

    private void HandlePause() => _model.RequestPause();

    public void Dispose()
    {
        _model.OnScoreChanged  -= _view.SetScore;
        _model.OnHealthChanged -= _view.SetHealthRatio;
        _view.OnPauseClicked   -= HandlePause;
    }
}
```

**원칙:**
- View는 Model을 직접 참조하지 않는다
- Presenter는 Unity API(`MonoBehaviour` 등)에 의존하지 않는다
- 이벤트 구독은 생성자에서, 해제는 반드시 `Dispose`에서

---

## 팝업 / 동적 UI 패턴

팝업은 Prefab이 아닌 `VisualTreeAsset.Instantiate()`로 생성한다.  
팝업을 자주 생성/파괴한다면 풀링을 적용할 것 (`references/performance.md` 참조).

```csharp
public class PopupService
{
    private readonly VisualElement _overlayRoot;
    private readonly VisualTreeAsset _popupTemplate;

    public void ShowPopup(string title, string message)
    {
        var popup = _popupTemplate.Instantiate();
        popup.Q<Label>("popup-title").text = title;
        popup.Q<Label>("popup-message").text = message;
        popup.Q<Button>("popup-close").clicked += () => ClosePopup(popup);

        _overlayRoot.Add(popup);

        // 다음 프레임에 클래스 추가 → USS transition 발동
        popup.schedule.Execute(() =>
            popup.AddToClassList("popup--visible")).StartingIn(0);
    }

    private void ClosePopup(VisualElement popup)
    {
        popup.RemoveFromClassList("popup--visible");
        // USS transition duration(--duration-slow)과 맞춰서 제거
        popup.schedule.Execute(() => popup.RemoveFromHierarchy())
            .StartingIn(350);
    }
}
```

```css
/* popup.uss */
.popup {
    opacity: 0;
    translate: 0 16px;
    transition: opacity var(--duration-normal) var(--easing-default),
                translate var(--duration-normal) var(--easing-default);
}

.popup--visible {
    opacity: 1;
    translate: 0 0;
}
```

---

## 에디터 UI

EditorWindow / CustomInspector도 동일하게 UI Toolkit으로 구현한다.  
런타임 UI와 맥락이 다르므로 구현 요청 시 별도로 안내한다.

**핵심 차이점만 기억할 것:**
- UXML/USS 로드는 `AssetDatabase.LoadAssetAtPath<T>()` 사용
- 진입점은 `CreateGUI()` 오버라이드
- `UnityEditor.UIElements` 네임스페이스 추가 필요

---

## 자주 하는 실수 / 금지 패턴

| ❌ 하지 말 것 | ✅ 대신 할 것 |
|---|---|
| `GameObject.Find()` + `GetComponent<Image>()` | `root.Q<VisualElement>("name")` |
| `Update()`에서 매 프레임 `Q<>()` 호출 | `Awake()`에서 한 번만 캐싱 |
| `clicked += () => ...` 람다로 등록 후 `-=` 해제 시도 | 메서드 참조로 등록 후 동일 참조로 해제 |
| USS 없이 C#에서 인라인 스타일 남발 | USS 클래스 토글 (`EnableInClassList`) |
| 색상·간격 값 하드코딩 | `Variables.uss` 토큰 사용 |
| `Canvas/RectTransform` 사용 | `UIDocument` + UXML |
| `Text` 또는 `TMP_Text` 컴포넌트 | `Label` VisualElement |
| 팝업을 `Instantiate(prefab)`으로 생성 | `VisualTreeAsset.Instantiate()` |
| `:hover`만 작성하고 `:focus` 누락 | 게임패드 대응을 위해 `:focus`도 함께 작성 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| HealthBar처럼 재사용 가능한 커스텀 컨트롤 만들기, 드래그 같은 커스텀 입력 처리(Manipulator) | `references/custom-controls.md` |
| ListView 아이템 바인딩, Unity 6 공식 바인딩 API, ScrollView 동적 콘텐츠 | `references/data-binding.md` |
| 아이템이 많아서 느릴 때, 팝업 풀링, USS transition vs C# schedule 선택, 프로파일링 방법 | `references/performance.md` |

공식 예제 저장소: https://github.com/Unity-Technologies/ui-toolkit-manual-code-examples
