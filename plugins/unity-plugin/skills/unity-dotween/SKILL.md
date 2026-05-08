---
name: unity-dotween
description: >
  DOTween(트위닝 라이브러리)을 사용하는 작업에서 반드시 참조하는 스킬.
  올바른 트윈 생성, Sequence 조합, 수명 관리 방식을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - DOTween, DOMove, DORotate, DOScale, DOFade 등 트윈 관련 작업
  - Sequence, Append, Join, Insert 등 트윈 조합 작업
  - SetEase, SetLoops, OnComplete 등 트윈 설정/콜백 작업
  - 보간 애니메이션, 트위닝, 이징 관련 작업
  - Punch, Shake 등 피드백 효과

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - DOTween을 사용하지 않는 프로젝트
  - Animator/Animation 컴포넌트로 처리하는 애니메이션
  - USS transition으로 충분한 UI Toolkit 애니메이션 (unity-uitoolkit 스킬 참조)
user-invocable: false
---

# Unity DOTween Skill

## 핵심 원칙

> **트윈은 반드시 수명을 관리한다.**
> `SetLink(gameObject)`로 GameObject에 연결하거나, 수동으로 `Kill()`한다.
> 관리되지 않는 트윈은 대상 오브젝트 파괴 후 에러를 발생시킨다.

> **Sequence를 적극 활용한다.**
> 여러 트윈을 조합할 때 코루틴 대신 Sequence를 사용한다.
> `Append`(순차), `Join`(동시), `Insert`(특정 시점)로 타임라인을 구성한다.

공식 문서: https://dotween.demigiant.com/documentation.php

---

## Tweener 생성

### Transform 이동

```csharp
// 월드 좌표 이동
transform.DOMove(new Vector3(5, 0, 0), 1f);

// 단일 축 이동
transform.DOMoveX(5f, 1f);
transform.DOMoveY(3f, 1f);

// 로컬 좌표 이동
transform.DOLocalMove(new Vector3(0, 2, 0), 1f);

// 점프
transform.DOJump(targetPos, jumpPower: 2f, numJumps: 1, duration: 0.5f);
```

### Transform 회전

```csharp
// 오일러 각도 회전
transform.DORotate(new Vector3(0, 180, 0), 1f);

// 회전 모드
transform.DORotate(new Vector3(0, 360, 0), 1f, RotateMode.FastBeyond360);

// 로컬 회전
transform.DOLocalRotate(new Vector3(0, 90, 0), 0.5f);

// 대상을 바라보기
transform.DOLookAt(target.position, 0.5f);
```

**RotateMode:**

| 모드 | 동작 |
|---|---|
| `Fast` (기본) | 최단 경로, 360° 이내 |
| `FastBeyond360` | 360° 초과 회전 허용 |
| `WorldAxisAdd` | 월드 축 기준 누적 회전 |
| `LocalAxisAdd` | 로컬 축 기준 누적 회전 |

### Transform 스케일

```csharp
// 균일 스케일
transform.DOScale(2f, 0.5f);

// 벡터 스케일
transform.DOScale(new Vector3(2, 1, 1), 0.5f);

// 단일 축
transform.DOScaleX(0f, 0.3f);
```

### 색상 / 알파

```csharp
// Material 색상
renderer.material.DOColor(Color.red, 0.5f);

// Material 알파
renderer.material.DOFade(0f, 0.5f);

// SpriteRenderer
spriteRenderer.DOColor(Color.red, 0.3f);
spriteRenderer.DOFade(0f, 0.3f);

// UI Image
image.DOColor(Color.green, 0.5f);
image.DOFade(0f, 0.5f);
image.DOFillAmount(1f, 1f);

// CanvasGroup
canvasGroup.DOFade(0f, 0.3f);

// Light
light.DOColor(Color.yellow, 0.5f);
light.DOIntensity(2f, 0.5f);
```

### 범용 트윈 (DOTween.To)

어떤 값이든 트윈할 수 있다.

```csharp
// float 트윈
float myValue = 0;
DOTween.To(() => myValue, x => myValue = x, 100f, 1f);

// Vector3 트윈
Vector3 myPos = Vector3.zero;
DOTween.To(() => myPos, x => myPos = x, new Vector3(5, 5, 0), 1f);

// Color 알파 트윈
DOTween.ToAlpha(() => myColor, x => myColor = x, 0f, 0.5f);
```

### FROM 트윈

`.From()`을 체인하면 시작값과 끝값이 반전된다.

```csharp
// 현재 위치 → targetPos (일반)
transform.DOMove(targetPos, 1f);

// targetPos → 현재 위치 (FROM)
transform.DOMove(targetPos, 1f).From();
```

---

## Sequence (트윈 조합)

### 기본 구조

```csharp
var seq = DOTween.Sequence();

// 순차 실행 (하나 끝나면 다음)
seq.Append(transform.DOMoveX(5f, 1f));
seq.Append(transform.DORotate(new Vector3(0, 180, 0), 0.5f));

// 동시 실행 (마지막 Append와 같은 시점)
seq.Append(transform.DOMoveX(5f, 1f));
seq.Join(transform.DOScale(2f, 1f));  // DOMoveX와 동시 실행

// 특정 시점에 삽입
seq.Insert(0.5f, transform.DOFade(0f, 0.5f));  // 0.5초 시점에 시작

// 대기 시간 삽입
seq.AppendInterval(0.5f);

// 콜백 삽입
seq.AppendCallback(() => Debug.Log("중간 지점"));
seq.InsertCallback(1f, () => PlaySound());
```

### Sequence 조합 메서드

| 메서드 | 동작 |
|---|---|
| `Append(tween)` | 끝에 순차 추가 |
| `Join(tween)` | 마지막 Append와 동시 실행 |
| `Insert(time, tween)` | 특정 시점에 삽입 |
| `Prepend(tween)` | 처음에 추가 (나머지 뒤로 밀림) |
| `AppendInterval(time)` | 대기 시간 추가 |
| `AppendCallback(action)` | 콜백 추가 |
| `InsertCallback(time, action)` | 특정 시점에 콜백 |

### Sequence 예시: UI 등장 애니메이션

```csharp
public Sequence CreateShowAnimation(CanvasGroup group, RectTransform rect)
{
    // 초기 상태 설정
    group.alpha = 0f;
    rect.anchoredPosition = new Vector2(0, -50f);

    var seq = DOTween.Sequence();
    seq.Append(group.DOFade(1f, 0.3f));
    seq.Join(rect.DOAnchorPosY(0f, 0.3f).SetEase(Ease.OutBack));
    seq.SetLink(group.gameObject);

    return seq;
}
```

### Sequence 예시: 순차 이벤트

```csharp
var seq = DOTween.Sequence();
seq.Append(transform.DOMove(pointA, 1f));
seq.AppendCallback(() => PlayEffect("arrive"));
seq.AppendInterval(0.5f);
seq.Append(transform.DOMove(pointB, 1f));
seq.Append(transform.DOScale(0f, 0.3f));
seq.AppendCallback(() => Destroy(gameObject));
```

---

## 설정 (Set 체인)

### 이징

```csharp
transform.DOMove(target, 1f).SetEase(Ease.OutBounce);
transform.DOScale(2f, 0.5f).SetEase(Ease.OutBack);
canvasGroup.DOFade(1f, 0.3f).SetEase(Ease.OutQuad);
```

**주요 Ease 타입:**

| Ease | 특성 | 용도 |
|---|---|---|
| `Linear` | 일정 속도 | 진행 바, 타이머 |
| `OutQuad` | 부드러운 감속 | 일반적인 이동 (기본값) |
| `OutBack` | 약간 넘어갔다 돌아옴 | UI 등장, 스케일 |
| `OutBounce` | 바운스 효과 | 낙하, 착지 |
| `OutElastic` | 탄성 효과 | 강조, 타격 피드백 |
| `InOutQuad` | 양쪽 부드러움 | 카메라 이동 |
| `InBack` | 뒤로 갔다 출발 | UI 퇴장 |

AnimationCurve도 사용 가능:

```csharp
[SerializeField] private AnimationCurve _customCurve;
transform.DOMove(target, 1f).SetEase(_customCurve);
```

### 반복

```csharp
// 3회 반복
transform.DOMove(target, 1f).SetLoops(3, LoopType.Restart);

// 왕복 반복
transform.DOScale(1.2f, 0.5f).SetLoops(-1, LoopType.Yoyo);

// 누적 반복
transform.DOMoveX(2f, 1f).SetLoops(3, LoopType.Incremental);
```

**LoopType:**

| 타입 | 동작 |
|---|---|
| `Restart` | 처음부터 다시 |
| `Yoyo` | 끝 → 시작 → 끝 왕복 |
| `Incremental` | 차이값 누적 (Tweener 전용) |

### 지연

```csharp
transform.DOMove(target, 1f).SetDelay(0.5f);
```

### 상대값

```csharp
// 현재 위치에서 (3, 0, 0)만큼 이동
transform.DOMove(new Vector3(3, 0, 0), 1f).SetRelative();
```

### TimeScale 독립

```csharp
// Time.timeScale = 0 일 때도 실행
transform.DOMove(target, 1f).SetUpdate(true);

// UpdateType 명시
transform.DOMove(target, 1f).SetUpdate(UpdateType.Normal, isIndependentUpdate: true);
```

### 수명 연결 (SetLink)

```csharp
// GameObject 파괴 시 자동 Kill
transform.DOMove(target, 1f).SetLink(gameObject);

// 비활성화 시 일시정지, 활성화 시 재시작
transform.DOScale(1.2f, 0.5f)
    .SetLoops(-1, LoopType.Yoyo)
    .SetLink(gameObject, LinkBehaviour.PauseOnDisableRestartOnEnable);
```

### AutoKill 비활성화

```csharp
// 완료 후에도 유지 (재사용 가능)
var tween = transform.DOMove(target, 1f).SetAutoKill(false);

// 나중에 재시작
tween.Restart();
```

---

## 콜백 (On 체인)

```csharp
transform.DOMove(target, 1f)
    .OnStart(() => Debug.Log("시작"))
    .OnUpdate(() => Debug.Log("매 프레임"))
    .OnStepComplete(() => Debug.Log("루프 1회 완료"))
    .OnComplete(() => Debug.Log("전체 완료"))
    .OnKill(() => Debug.Log("Kill됨"));
```

| 콜백 | 시점 |
|---|---|
| `OnStart` | 첫 재생 시작 시 (한 번) |
| `OnPlay` | 재생/재개 시 |
| `OnPause` | 일시정지 시 |
| `OnUpdate` | 매 프레임 |
| `OnStepComplete` | 루프 1회 완료 시 |
| `OnComplete` | 모든 루프 완료 시 |
| `OnRewind` | 되감기 시 |
| `OnKill` | Kill 시 |

---

## 제어 메서드

```csharp
tween.Play();            // 재생
tween.Pause();           // 일시정지
tween.Kill();            // 제거
tween.Kill(complete: true); // 완료 후 제거
tween.Restart();         // 처음부터 재시작
tween.Rewind();          // 처음으로 되감기 (재생 안 함)
tween.PlayForward();     // 정방향 재생
tween.PlayBackwards();   // 역방향 재생
tween.Complete();        // 즉시 완료
tween.Goto(0.5f, andPlay: true);  // 특정 시점으로 이동
```

### 상태 확인

```csharp
tween.IsActive();        // 활성 상태인지
tween.IsPlaying();       // 재생 중인지
tween.IsComplete();      // 완료되었는지
tween.Duration();        // 전체 시간
tween.Elapsed();         // 경과 시간
tween.CompletedLoops();  // 완료된 루프 수
```

---

## 피드백 효과

### Punch (충격)

지정 방향으로 튀었다 돌아온다.

```csharp
// 위치 펀치 (HIT 피드백)
transform.DOPunchPosition(new Vector3(0, 0.5f, 0), 0.3f, vibrato: 5, elasticity: 0.5f);

// 회전 펀치
transform.DOPunchRotation(new Vector3(0, 0, 15f), 0.3f, vibrato: 5);

// 스케일 펀치
transform.DOPunchScale(Vector3.one * 0.3f, 0.3f, vibrato: 3);
```

### Shake (진동)

랜덤 방향으로 흔든다.

```csharp
// 위치 흔들기 (카메라 흔들림)
transform.DOShakePosition(0.5f, strength: 0.5f, vibrato: 10, randomness: 90f);

// 회전 흔들기
transform.DOShakeRotation(0.3f, strength: 30f, vibrato: 10);

// 카메라 흔들기
camera.DOShakePosition(0.3f, strength: 0.3f, vibrato: 10);
```

---

## UniTask 연동

`UNITASK_DOTWEEN_SUPPORT` 스크립팅 심볼 정의 필요.

```csharp
// 트윈 완료 대기
await transform.DOMove(target, 1f).WithCancellation(ct);

// 병렬 트윈 대기
await UniTask.WhenAll(
    transform.DOMove(target, 1f).WithCancellation(ct),
    transform.DORotate(rotation, 1f).WithCancellation(ct));

// Sequence 완료 대기
var seq = DOTween.Sequence()
    .Append(transform.DOMoveX(5f, 1f))
    .Append(transform.DOScale(2f, 0.5f));
await seq.WithCancellation(ct);
```

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| `SetLink` 없이 트윈 생성 | `SetLink(gameObject)`로 수명 연결 |
| Sequence에 이미 시작된 트윈 추가 | 트윈 생성 직후 Sequence에 추가 |
| Sequence 안에 무한 루프 트윈 | 루트 Sequence에만 무한 루프 설정 |
| `SetAutoKill(false)` 후 Kill 누락 | OnDestroy에서 `tween?.Kill()` |
| 매 프레임 새 트윈 생성 | 한 번 생성 후 재사용 또는 조건 체크 |
| `DORotate`에 Quaternion 전달 | Vector3(오일러) 사용, Quaternion은 `DORotateQuaternion` |
| `async void`에서 트윈 await | `async UniTask`에서 `.WithCancellation(ct)` |
| 코루틴으로 트윈 순차 실행 | Sequence 사용 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| 컴포넌트별 숏컷 전체 목록, Path 트윈, Blendable 트윈 | `references/shortcuts.md` |
| UI 애니메이션 패턴, 게임 피드백 패턴, 실전 Sequence 조합 | `references/patterns.md` |
