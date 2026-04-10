# Unity UI Toolkit — 데이터 바인딩 참조

## Unity 6+ 공식 런타임 데이터 바인딩

Unity 6부터 `INotifyBindablePropertyChanged` 기반의 공식 런타임 바인딩 API가 제공된다.

### 기본 바인딩 패턴

```csharp
// 바인딩 가능한 데이터 소스
public class PlayerData : INotifyBindablePropertyChanged
{
    public event EventHandler<BindablePropertyChangedEventArgs> propertyChanged;

    private int _score;
    [CreateProperty]
    public int Score
    {
        get => _score;
        set
        {
            if (_score == value) return;
            _score = value;
            propertyChanged?.Invoke(this,
                new BindablePropertyChangedEventArgs(nameof(Score)));
        }
    }
}

// View에서 바인딩 연결
var root = uiDocument.rootVisualElement;
root.dataSource = playerData;

var scoreLabel = root.Q<Label>("score-label");
scoreLabel.SetBinding("text",
    new DataBinding { dataSourcePath = new PropertyPath(nameof(PlayerData.Score)) });
```

---

## 수동 바인딩 (Presenter 패턴)

Unity 6 미만이거나 공식 바인딩 API 없이 Presenter에서 직접 연결하는 방식.  
순수 C# 이벤트로 작성하며, Rx 라이브러리가 있다면 Subscribe로 대체할 수 있다.

```csharp
// Model: 상태 변화를 이벤트로 노출
public class GameStateModel
{
    public event Action<int>   OnScoreChanged;
    public event Action<float> OnHealthChanged;

    private int _score;
    public int Score
    {
        get => _score;
        set { _score = value; OnScoreChanged?.Invoke(_score); }
    }

    private float _healthRatio;
    public float HealthRatio
    {
        get => _healthRatio;
        set { _healthRatio = value; OnHealthChanged?.Invoke(_healthRatio); }
    }
}

// Presenter: Model 이벤트 → View 업데이트
public class HudPresenter : IDisposable
{
    private readonly HudView _view;
    private readonly GameStateModel _model;

    public HudPresenter(HudView view, GameStateModel model)
    {
        _view  = view;
        _model = model;

        _model.OnScoreChanged  += _view.SetScore;
        _model.OnHealthChanged += _view.SetHealthRatio;
    }

    public void Dispose()
    {
        _model.OnScoreChanged  -= _view.SetScore;
        _model.OnHealthChanged -= _view.SetHealthRatio;
    }
}
```

### 양방향 바인딩 (입력 필드)

```csharp
// View: 입력 변경 이벤트 노출
public class SettingsView : MonoBehaviour
{
    private TextField _nameField;
    public event Action<string> OnNameChanged;

    private void Awake()
    {
        _nameField = _uiDocument.rootVisualElement.Q<TextField>("name-field");
        _nameField.RegisterValueChangedCallback(e => OnNameChanged?.Invoke(e.newValue));
    }
}

// Presenter: View 이벤트 → Model 반영
_view.OnNameChanged += name => _model.SetPlayerName(name);
```

---

## ListView 바인딩 패턴

```csharp
// 아이템 리스트 바인딩 — makeItem/bindItem 패턴 필수
var listView = root.Q<ListView>("item-list");
listView.makeItem = () => new Label();
listView.bindItem = (element, index) =>
{
    var label = (Label)element;
    label.text = items[index].Name;
};
listView.itemsSource = items;  // IList
listView.Rebuild();

// 선택 이벤트
listView.selectionChanged += selection =>
{
    var item = (ItemData)selection.First();
    OnItemSelected?.Invoke(item);
};
```

---

## ScrollView 동적 콘텐츠

```csharp
var scrollView = root.Q<ScrollView>("content-scroll");

foreach (var data in dataList)
{
    var itemElement = _itemTemplate.Instantiate();
    itemElement.Q<Label>("item-name").text = data.Name;
    scrollView.contentContainer.Add(itemElement);
}

// 맨 아래로 스크롤
scrollView.schedule.Execute(() =>
    scrollView.ScrollTo(scrollView.contentContainer.ElementAt(
        scrollView.contentContainer.childCount - 1)));
```

---

## 참고 예제 (공식 저장소)
- `bind-custom-control/` : 커스텀 컨트롤 바인딩
- `ListViewExample/` : ListView 완전 예제
- URL: https://github.com/Unity-Technologies/ui-toolkit-manual-code-examples
