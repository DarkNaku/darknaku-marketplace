# 카메라 패턴 상세

## 2D 카메라 구성

### 기본 2D 설정

```csharp
// 1. Cinemachine > Create Virtual Camera
// 2. Body → Framing Transposer
// 3. Aim → Do Nothing (2D에서는 회전 불필요)
// 4. Follow에 플레이어 할당
// 5. Camera Distance: 직교 카메라에서는 실제 거리는 무관하나 기본값 유지
```

### Framing Transposer 2D 설정

```
Screen X/Y: 0.5 (화면 중앙)
Dead Zone Width: 0.05 ~ 0.15 (작은 여유 공간)
Dead Zone Height: 0.05 ~ 0.15
Soft Zone Width: 0.6 ~ 0.8
Soft Zone Height: 0.6 ~ 0.8
X Damping: 1 ~ 2
Y Damping: 1 ~ 2
Camera Distance: 10 (Z 오프셋)
```

### 2D 화면 영역 시각화

```
┌─────────────────────────────────────────┐
│                Soft Zone                │ ← 카메라가 부드럽게 추적
│    ┌───────────────────────────┐        │
│    │        Dead Zone          │        │ ← 카메라 이동 없음
│    │                           │        │
│    │         ● 타겟             │        │
│    │                           │        │
│    └───────────────────────────┘        │
│                                         │
└─────────────────────────────────────────┘
← Hard Limit 밖으로 나가면 즉시 추적 →
```

### Dead Zone 크기별 효과

| 크기 | 효과 | 적합한 장르 |
|---|---|---|
| 0 (없음) | 타겟을 정확히 화면 중앙 유지 | 탑뷰, RTS |
| 0.05 ~ 0.1 | 미세한 여유. 거의 중앙 유지 | 정밀 플랫포머 |
| 0.1 ~ 0.2 | 적당한 여유. 자연스러운 추적 | 일반 2D 게임 |
| 0.3+ | 넓은 여유. 느슨한 추적 | 탐색 중심 게임 |

---

## Confiner 상세

### 2D Confiner 설정

```csharp
// 1. 빈 GameObject 생성 → PolygonCollider2D 추가 → Is Trigger 체크
// 2. PolygonCollider2D의 포인트를 편집하여 카메라 영역 정의
// 3. Virtual Camera → Add Extension → CinemachineConfiner
// 4. Bounding Shape 2D에 PolygonCollider2D 할당
// 5. Confine Screen Edges: true (직교 카메라 권장)
```

### 3D Confiner 설정

```csharp
// 1. 빈 GameObject 생성 → BoxCollider 추가 → Is Trigger 체크
// 2. BoxCollider 크기 조정
// 3. Virtual Camera → Add Extension → CinemachineConfiner
// 4. Bounding Volume에 Collider 할당
// 5. Damping: 0.5 ~ 1 (부드러운 복귀)
```

### Confine Screen Edges

```
Confine Screen Edges = false:
  → 카메라 중심만 영역 안에 유지
  → 화면 가장자리가 영역 밖으로 나갈 수 있음

Confine Screen Edges = true:
  → 전체 화면이 영역 안에 유지
  → 직교(Orthographic) 카메라에서 권장
  → 화면 가장자리가 콜라이더 밖으로 나가지 않음
```

### 방 전환 패턴

```csharp
// 방(Room) 단위 Confiner 전환

[SerializeField] private CinemachineVirtualCamera _cam;
[SerializeField] private CinemachineConfiner _confiner;

public void ChangeRoom(Collider2D newRoomBounds)
{
    _confiner.m_BoundingShape2D = newRoomBounds;
    _confiner.InvalidatePathCache();  // 경계 캐시 갱신 필수!
}
```

---

## 3인칭 카메라 패턴

### 기본 3인칭 (Transposer + Composer)

```
Body → Transposer
  Binding Mode: Lock To Target With World Up
  Follow Offset: (0, 2, -5) 또는 (1, 2, -5) 어깨 위
  X/Y/Z Damping: 1

Aim → Composer
  Tracked Object Offset: (0, 1.5, 0) — 머리 위
  Dead Zone Width/Height: 0.05
  Horizontal/Vertical Damping: 0.5
```

### 3인칭 오비탈 (FreeLook)

```
Top Rig:    Height = 4.5, Radius = 2
Middle Rig: Height = 2.5, Radius = 5
Bottom Rig: Height = 0.4, Radius = 2

X Axis > Max Speed: 300
Y Axis > Max Speed: 2
Spline Curvature: 0.5

각 리그의 Aim → Composer:
  Tracked Object Offset: (0, 1, 0)
  Dead Zone: 0.1 x 0.1
```

### 어깨 너머 카메라 (Over-the-Shoulder)

```
Body → Transposer
  Follow Offset: (1.5, 1.5, -3)  — 오른쪽 어깨 위
  Binding Mode: Lock To Target With World Up

Aim → Composer
  Screen X: 0.35 (왼쪽으로 치우침 → 오른쪽에 공간)
  Screen Y: 0.4 (약간 위)
  Dead Zone Width: 0.05
  Dead Zone Height: 0.05
```

---

## FPS 카메라 패턴

```
Body → Hard Lock to Target
  (카메라가 타겟 위치에 정확히 일치)

Aim → POV
  Vertical Axis:
    Value Range: -70 ~ 70  (위아래 제한)
    Max Speed: 300
    Input Axis Name: "Mouse Y"
    Invert: true

  Horizontal Axis:
    Value Range: -180 ~ 180
    Wrap: true  (360° 회전)
    Max Speed: 300
    Input Axis Name: "Mouse X"
    Invert: false

  Recentering: 비활성화 (FPS에서는 자동 복귀 불필요)
```

---

## 탑뷰 / RTS 카메라 패턴

```
Body → Transposer
  Binding Mode: World Space
  Follow Offset: (0, 20, -10)  — 위에서 비스듬히
  X/Y/Z Damping: 0.5

Aim → Hard Look At
  (항상 타겟을 정중앙에 고정)

// RTS: Follow 타겟을 빈 GameObject로 만들고
// WASD/마우스로 해당 오브젝트를 이동시켜 카메라 패닝 구현
```

---

## 다중 타겟 (Target Group) 패턴

### 보스전: 플레이어 + 보스 동시 프레이밍

```csharp
[SerializeField] private CinemachineTargetGroup _bossTargetGroup;
[SerializeField] private CinemachineVirtualCamera _bossCam;

public void StartBossFight(Transform boss, Transform player)
{
    _bossTargetGroup.AddMember(player, weight: 1f, radius: 1f);
    _bossTargetGroup.AddMember(boss, weight: 1f, radius: 3f);
    _bossCam.Priority = 20;
}

public void EndBossFight()
{
    _bossCam.Priority = 0;
}
```

### 멀티플레이어 분할 화면 없이

```csharp
// 모든 플레이어를 TargetGroup에 추가
// Group Composer가 FOV 또는 거리를 자동 조정하여 모두 프레임에 유지
// Group Framing Mode: Horizontal And Vertical
// Adjustment Mode: Zoom Only (FOV 변경) 또는 Dolly Only (카메라 이동)
// Group Framing Size: 0.8 (화면의 80% 차지)
```

---

## Noise 활용 패턴

### 상태별 핸드헬드 흔들림

```csharp
[SerializeField] private CinemachineVirtualCamera _cam;
[SerializeField] private NoiseSettings _idleNoise;     // 미세한 흔들림
[SerializeField] private NoiseSettings _runNoise;      // 달리기 흔들림
[SerializeField] private NoiseSettings _sprintNoise;   // 전력질주 흔들림

private CinemachineBasicMultiChannelPerlin _noise;

private void Awake()
{
    _noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
}

public void SetMovementState(MovementState state)
{
    (_noise.m_NoiseProfile, _noise.m_AmplitudeGain, _noise.m_FrequencyGain) = state switch
    {
        MovementState.Idle    => (_idleNoise, 0.3f, 0.5f),
        MovementState.Run     => (_runNoise, 0.6f, 1f),
        MovementState.Sprint  => (_sprintNoise, 1f, 1.5f),
        _                     => (_idleNoise, 0f, 0f),
    };
}
```

### 일시적 카메라 흔들림 (코루틴)

```csharp
public IEnumerator ShakeCoroutine(float duration, float amplitude)
{
    var noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
    noise.m_AmplitudeGain = amplitude;

    yield return new WaitForSeconds(duration);

    noise.m_AmplitudeGain = 0f;
}
```

### 일시적 카메라 흔들림 (UniTask)

```csharp
public async UniTask ShakeAsync(float duration, float amplitude, CancellationToken ct)
{
    var noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
    noise.m_AmplitudeGain = amplitude;

    await UniTask.Delay(TimeSpan.FromSeconds(duration), cancellationToken: ct);

    noise.m_AmplitudeGain = 0f;
}
```

---

## Collider (장애물 회피) 상세

### Strategy 비교

| Strategy | 동작 | 적합한 상황 |
|---|---|---|
| `Pull Camera Forward` | 장애물 앞으로 카메라 당김 | 3인칭 일반 |
| `Preserve Camera Height` | 높이 유지하며 회피 | 탑뷰 계열 |
| `Preserve Camera Distance` | 거리 유지하며 회피 | 오비탈 카메라 |

### 설정 권장값

```
Collide Against: Default, Environment (장애물 레이어)
Avoid Obstacles: true
Camera Radius: 0.3 ~ 0.5
Strategy: Pull Camera Forward (3인칭 기본)
Smoothing Time: 0.2
Damping: 1
Damping When Occluded: 0.5 (빠르게 회피)
Optimal Target Distance: 0 (사용 안 함)
```

---

## Storyboard Extension

카메라 위에 참조 이미지를 오버레이하여 구도 확인.

```
Virtual Camera → Add Extension → CinemachineStoryboard
Image: 참조 이미지 할당
Alpha: 0.3 ~ 0.5 (반투명)
Aspect: Best Fit
Split View: 좌우 분할 비교
Mute Camera: true → 카메라 업데이트 중지 (구도 고정)
```
