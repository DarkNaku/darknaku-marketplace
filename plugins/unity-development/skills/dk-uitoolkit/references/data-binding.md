# Unity UI Toolkit — 데이터 바인딩 참조

## Unity 6+ 공식 런타임 데이터 바인딩

Unity 6부터 `INotifyValueChanged<T>` 기반의 공식 런타임 바인딩 API가 제공된다.

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

### R3 + 수동 바인딩 (현재 프로젝트 권장)

Unity 6 미만이거나 R3를 이미 사용 중이면 Presenter에서 수동으로 바인딩하는 게 더 일관성 있다.

```csharp
// R3 ReactiveProperty → View 업데이트
_model.Score
    .Subscribe(score => _view.SetScore(score))
    .AddTo(_disposables);

// 양방향 바인딩 (입력 필드)
_view.OnNameChanged
    .Subscribe(name => _model.SetPlayerName(name))
    .AddTo(_disposables);
```

### ListView 바인딩 패턴

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

### ScrollView 동적 콘텐츠

```csharp
var scrollView = root.Q<ScrollView>("content-scroll");

// 아이템 추가
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
