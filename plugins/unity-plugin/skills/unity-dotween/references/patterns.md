# DOTween 실전 패턴

## UI 애니메이션 패턴

### 팝업 등장/퇴장

```csharp
public class PopupAnimator
{
    private readonly CanvasGroup _canvasGroup;
    private readonly RectTransform _rectTransform;

    public Sequence ShowPopup()
    {
        _canvasGroup.alpha = 0f;
        _rectTransform.localScale = Vector3.one * 0.8f;

        var seq = DOTween.Sequence();
        seq.Append(_canvasGroup.DOFade(1f, 0.25f));
        seq.Join(_rectTransform.DOScale(1f, 0.3f).SetEase(Ease.OutBack));
        seq.SetLink(_canvasGroup.gameObject);
        return seq;
    }

    public Sequence HidePopup()
    {
        var seq = DOTween.Sequence();
        seq.Append(_canvasGroup.DOFade(0f, 0.2f));
        seq.Join(_rectTransform.DOScale(0.8f, 0.2f).SetEase(Ease.InBack));
        seq.SetLink(_canvasGroup.gameObject);
        return seq;
    }
}
```

### 리스트 아이템 순차 등장

```csharp
public Sequence AnimateListItems(RectTransform[] items, float stagger = 0.05f)
{
    var seq = DOTween.Sequence();

    for (int i = 0; i < items.Length; i++)
    {
        var item = items[i];
        var cg = item.GetComponent<CanvasGroup>();
        cg.alpha = 0f;
        item.anchoredPosition += new Vector2(0, -30f);

        seq.Insert(i * stagger,
            cg.DOFade(1f, 0.2f));
        seq.Insert(i * stagger,
            item.DOAnchorPosY(item.anchoredPosition.y + 30f, 0.3f)
                .SetEase(Ease.OutQuad));
    }

    return seq;
}
```

### 버튼 프레스 피드백

```csharp
public void PlayButtonPress(Transform buttonTransform)
{
    buttonTransform.DOPunchScale(Vector3.one * -0.1f, 0.15f, vibrato: 5, elasticity: 0.5f)
        .SetLink(buttonTransform.gameObject);
}
```

### 탭 전환 슬라이드

```csharp
public Sequence SwitchTab(RectTransform currentTab, RectTransform nextTab, bool slideLeft)
{
    float direction = slideLeft ? -1f : 1f;
    float screenWidth = Screen.width;

    nextTab.anchoredPosition = new Vector2(screenWidth * -direction, 0);

    var seq = DOTween.Sequence();
    seq.Append(currentTab.DOAnchorPosX(screenWidth * direction, 0.3f).SetEase(Ease.InOutQuad));
    seq.Join(nextTab.DOAnchorPosX(0, 0.3f).SetEase(Ease.InOutQuad));
    return seq;
}
```

### HP 바 애니메이션

```csharp
public void AnimateHealthBar(Image fillImage, float targetRatio)
{
    fillImage.DOFillAmount(targetRatio, 0.5f)
        .SetEase(Ease.OutQuad)
        .SetLink(fillImage.gameObject);

    // 위험 구간 색상 변경
    if (targetRatio < 0.3f)
    {
        fillImage.DOColor(Color.red, 0.3f).SetLink(fillImage.gameObject);
    }
}
```

### 텍스트 타이핑 효과

```csharp
public Tween TypeText(Text textComponent, string fullText, float duration)
{
    textComponent.text = "";
    return textComponent.DOText(fullText, duration, richTextEnabled: true)
        .SetEase(Ease.Linear)
        .SetLink(textComponent.gameObject);
}
```

---

## 게임 피드백 패턴

### 데미지 히트 피드백

```csharp
public Sequence PlayHitFeedback(Transform target, SpriteRenderer sprite)
{
    var seq = DOTween.Sequence();

    // 위치 펀치
    seq.Append(target.DOPunchPosition(new Vector3(0.2f, 0, 0), 0.2f, vibrato: 10));

    // 피격 색상 플래시
    seq.Join(sprite.DOColor(Color.red, 0.1f)
        .SetLoops(2, LoopType.Yoyo));

    seq.SetLink(target.gameObject);
    return seq;
}
```

### 카메라 흔들림

```csharp
public void ShakeCamera(Camera cam, float intensity = 0.3f)
{
    cam.DOShakePosition(0.3f,
        strength: intensity,
        vibrato: 15,
        randomness: 90f,
        fadeOut: true)
        .SetLink(cam.gameObject);
}
```

### 아이템 획득 이펙트

```csharp
public Sequence PlayPickupEffect(Transform item)
{
    var seq = DOTween.Sequence();
    seq.Append(item.DOScale(1.5f, 0.15f).SetEase(Ease.OutQuad));
    seq.Append(item.DOScale(0f, 0.2f).SetEase(Ease.InBack));
    seq.Join(item.DOMoveY(item.position.y + 1f, 0.2f));
    seq.AppendCallback(() => Object.Destroy(item.gameObject));
    return seq;
}
```

### 적 스폰 등장

```csharp
public Sequence PlaySpawnAnimation(Transform enemy)
{
    enemy.localScale = Vector3.zero;

    var seq = DOTween.Sequence();
    seq.Append(enemy.DOScale(1.2f, 0.3f).SetEase(Ease.OutBack));
    seq.Append(enemy.DOScale(1f, 0.1f));
    seq.SetLink(enemy.gameObject);
    return seq;
}
```

### 코인 점프 수집

```csharp
public Sequence PlayCoinCollect(Transform coin, Transform target)
{
    var seq = DOTween.Sequence();
    seq.Append(coin.DOJump(target.position, jumpPower: 2f, numJumps: 1, duration: 0.5f));
    seq.Join(coin.DOScale(0f, 0.5f).SetEase(Ease.InQuad));
    seq.AppendCallback(() => Object.Destroy(coin.gameObject));
    return seq;
}
```

---

## 수명 관리 패턴

### MonoBehaviour에서의 관리

```csharp
public class EnemyView : MonoBehaviour
{
    private Tween _idleTween;

    void Start()
    {
        // 무한 루프 트윈 — SetLink으로 수명 관리
        _idleTween = transform.DOScale(1.05f, 1f)
            .SetLoops(-1, LoopType.Yoyo)
            .SetEase(Ease.InOutSine)
            .SetLink(gameObject);
    }

    public void PlayHit()
    {
        transform.DOPunchScale(Vector3.one * 0.2f, 0.2f)
            .SetLink(gameObject);
    }
}
```

### VContainer Presenter에서의 관리

```csharp
public class GamePresenter : IStartable, IDisposable
{
    private readonly GameView _view;
    private Sequence _introSequence;

    void IStartable.Start()
    {
        _introSequence = DOTween.Sequence()
            .Append(_view.Title.DOFade(1f, 1f))
            .AppendInterval(0.5f)
            .Append(_view.StartButton.DOScale(1f, 0.3f).SetEase(Ease.OutBack));
    }

    void IDisposable.Dispose()
    {
        _introSequence?.Kill();
    }
}
```

### SetAutoKill(false)로 재사용

```csharp
public class DoorController : MonoBehaviour
{
    private Tween _openTween;

    void Start()
    {
        _openTween = transform.DOLocalMoveY(3f, 0.5f)
            .SetAutoKill(false)
            .SetEase(Ease.OutQuad)
            .Pause();  // 자동 재생 방지
    }

    public void Open() => _openTween.PlayForward();
    public void Close() => _openTween.PlayBackwards();

    void OnDestroy() => _openTween?.Kill();
}
```

---

## Path 트윈

### 직선 경로

```csharp
Vector3[] waypoints = {
    new Vector3(0, 0, 0),
    new Vector3(5, 0, 0),
    new Vector3(5, 0, 5),
    new Vector3(0, 0, 5)
};

transform.DOPath(waypoints, 3f, PathType.Linear)
    .SetLoops(-1, LoopType.Restart)
    .SetEase(Ease.Linear)
    .SetLink(gameObject);
```

### 곡선 경로 (CatmullRom)

```csharp
transform.DOPath(waypoints, 3f, PathType.CatmullRom)
    .SetLookAt(0.01f)  // 이동 방향 바라보기
    .SetLink(gameObject);
```

---

## ID 기반 일괄 제어

```csharp
// ID 설정
transform.DOMove(target, 1f).SetId("enemyMove");
transform.DOScale(2f, 1f).SetId("enemyMove");

// ID로 일괄 제어
DOTween.Kill("enemyMove");
DOTween.Pause("enemyMove");
DOTween.Play("enemyMove");

// 대상 오브젝트로 제어
DOTween.Kill(transform);
DOTween.Pause(transform);
```
