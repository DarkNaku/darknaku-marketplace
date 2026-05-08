---
name: unity-cinemachine
description: >
  Cinemachine(카메라 시스템)을 사용하는 작업에서 반드시 참조하는 스킬.
  Virtual Camera 구성, Body/Aim 알고리즘 선택, 블렌딩, 카메라 흔들림 설정을 안내한다.

  다음 상황에서 반드시 이 스킬을 참조해야 한다:
  - CinemachineVirtualCamera, CinemachineFreeLook, CinemachineBrain 관련 작업
  - 카메라 추적(Follow), 조준(Look At), Body/Aim 알고리즘 설정
  - 카메라 블렌딩, 전환, 우선순위 기반 카메라 전환
  - 카메라 흔들림(Noise), Impulse 시스템, 충격 피드백
  - Confiner, Collider 등 Cinemachine Extension 설정
  - FreeLook, ClearShot, State-Driven, Mixing, Blend List 카메라
  - Cinemachine Path, Tracked Dolly 경로 기반 카메라 이동
  - Timeline과 Cinemachine 연동 (Cinemachine Track, Shot Clip)

  다음 상황에서는 이 스킬을 참조하지 않는다:
  - Cinemachine을 사용하지 않는 프로젝트
  - 단순 Camera 컴포넌트 직접 조작 (Cinemachine 미사용)
  - DOTween의 DOShakePosition 등 Cinemachine 외 카메라 흔들림 (unity-dotween 스킬 참조)
user-invocable: false
---

# Unity Cinemachine Skill

## 핵심 원칙

> **하나의 샷 = 하나의 Virtual Camera.**
> 카메라 앵글마다 별도의 Virtual Camera를 만든다.
> 코드로 카메라 속성을 직접 조작하지 않고, Virtual Camera 전환으로 해결한다.

> **Brain이 카메라를 제어한다.**
> Unity Camera에 CinemachineBrain을 부착하면 씬의 모든 Virtual Camera를 자동 관리한다.
> 가장 높은 Priority의 활성 Virtual Camera가 라이브 상태가 된다.

> **Body는 이동, Aim은 회전이다.**
> Body 알고리즘은 카메라 위치를, Aim 알고리즘은 카메라 회전을 제어한다.
> Follow 타겟과 Look At 타겟을 분리하여 유연한 카메라 동작을 구현한다.

공식 문서: https://docs.unity3d.com/kr/Packages/com.unity.cinemachine@2.3/manual/index.html

---

## 아키텍처 개요

```
Unity Camera (씬에 1개)
    └─ CinemachineBrain (컴포넌트)
         ├─ 모니터링 ──→ Virtual Camera 1 (Priority: 10, Live)
         ├─ 모니터링 ──→ Virtual Camera 2 (Priority: 5, Standby)
         └─ 모니터링 ──→ FreeLook Camera (Priority: 0, Standby)

Virtual Camera 구성:
    ├─ [Body]  → 카메라 위치 제어 (Transposer, Framing Transposer 등)
    ├─ [Aim]   → 카메라 회전 제어 (Composer, POV 등)
    ├─ [Noise] → 카메라 흔들림 (Basic Multi Channel Perlin)
    └─ [Extensions] → Confiner, Collider, PostProcessing 등
```

---

## CinemachineBrain

Unity Camera에 부착하는 컴포넌트. 씬의 모든 활성 Virtual Camera를 모니터링하고 라이브 카메라를 결정한다.

```csharp
// Brain은 직접 코드로 생성하지 않는다 — 에디터에서 추가
// Cinemachine > Create Virtual Camera 시 자동으로 Brain이 추가됨
```

### 주요 프로퍼티

| 프로퍼티 | 설명 |
|---|---|
| `Update Method` | Smart (기본), Fixed, Late Update 중 선택 |
| `Default Blend` | 카메라 전환 시 기본 블렌딩 (Ease In Out, Cut 등) |
| `Custom Blends` | 특정 카메라 쌍에 대한 커스텀 블렌딩 에셋 |
| `Ignore Time Scale` | `Time.timeScale = 0`에서도 동작 여부 |
| `World Up Override` | 월드 업 벡터 오버라이드 |
| `Camera Cut Event` | 카메라 컷 전환 시 발생 이벤트 |
| `Camera Activated Event` | 카메라 활성화 시 발생 이벤트 |

### Update Method 선택 기준

| 상황 | Update Method |
|---|---|
| 일반 게임플레이 | `Smart` (기본) |
| 물리 기반 이동 대상 | `Fixed Update` |
| 수동 제어 필요 | `Late Update` |

---

## Virtual Camera

가장 기본적인 Cinemachine 카메라. 실제 Unity 카메라를 생성하지 않고, Brain에 지시를 내린다.

### 주요 프로퍼티

| 프로퍼티 | 설명 |
|---|---|
| `Priority` | 높을수록 우선. 동일 시 최근 활성화된 카메라 |
| `Follow` | 카메라가 따라갈 타겟 |
| `Look At` | 카메라가 조준할 타겟 |
| `Lens > Field of View` | 수직 FOV |
| `Lens > Near/Far Clip` | 렌더링 범위 |
| `Lens > Dutch` | Z축 기울기 |
| `Position Blending` | Linear, Spherical, Cylindrical |

### 카메라 상태

| 상태 | 동작 |
|---|---|
| **Live** | Unity 카메라를 적극 제어 |
| **Standby** | 제어하지 않으나 타겟 추적, 매 프레임 업데이트 |
| **Disabled** | 제어하지 않고 타겟도 추적하지 않음 |

### 카메라 전환 (우선순위 기반)

```csharp
// 카메라 전환: Priority 변경 또는 GameObject 활성화/비활성화
[SerializeField] private CinemachineVirtualCamera _combatCam;
[SerializeField] private CinemachineVirtualCamera _exploreCam;

public void SwitchToCombat()
{
    _combatCam.Priority = 20;
    _exploreCam.Priority = 10;
}

// 또는 GameObject 활성화/비활성화
public void SwitchToExplore()
{
    _combatCam.gameObject.SetActive(false);
    _exploreCam.gameObject.SetActive(true);
}
```

---

## Body 알고리즘 (카메라 위치)

Follow 타겟 기준으로 카메라 위치를 결정하는 알고리즘.

### 알고리즘 선택 기준

| 알고리즘 | 용도 |
|---|---|
| `Transposer` | 3인칭 추적. 고정 오프셋으로 따라감 |
| `Framing Transposer` | 2D/사이드뷰. 화면 공간 기준 위치 유지 |
| `Orbital Transposer` | 3인칭 오비탈. 플레이어 입력으로 궤도 회전 |
| `Tracked Dolly` | 레일 카메라. 미리 정의된 경로를 따라 이동 |
| `Hard Lock to Target` | 1인칭. 타겟과 동일 위치 |
| `Do Nothing` | 위치 변경 없음 |

### Transposer

```csharp
// Inspector에서 설정
// Body → Transposer
// Follow Offset: (0, 5, -10)  — 타겟 뒤 위, 약간 뒤
// X/Y/Z Damping: 1  — 부드러운 추적
```

**Binding Mode:**

| 모드 | 동작 |
|---|---|
| `Lock To Target With World Up` | 요(yaw)만 추적. 3인칭 기본값 |
| `Lock To Target` | 모든 회전 추적 |
| `World Space` | 월드 공간 절대 오프셋 |
| `Simple Follow With World Up` | 카메라 로컬 공간 기준 |

### Framing Transposer (2D / 사이드뷰)

```csharp
// Inspector에서 설정
// Body → Framing Transposer
// Screen X/Y: 0.5 (화면 중앙)
// Camera Distance: 10
// Dead Zone Width/Height: 0.1 (작은 여유 공간)
// Soft Zone Width/Height: 0.8
// X/Y Damping: 1
```

**Lookahead:**

| 프로퍼티 | 설명 |
|---|---|
| `Lookahead Time` | 타겟 미래 위치 예측 시간 (초) |
| `Lookahead Smoothing` | 예측 부드러움 정도 |
| `Lookahead Ignore Y` | Y축 예측 무시 |

### Orbital Transposer (3인칭 오비탈)

```csharp
// Inspector에서 설정
// Body → Orbital Transposer
// Follow Offset: (0, 4, -8)
// X Axis > Input Axis Name: "Mouse X"
// Heading > Definition: Target Forward
// Recenter To Target Heading > Enabled: true
```

### Tracked Dolly (경로 카메라)

```csharp
// 1. Cinemachine Path 또는 Smooth Path 생성
// 2. Body → Tracked Dolly
// 3. Path에 경로 에셋 할당
// 4. Auto Dolly로 자동 경로 추적 또는 Path Position으로 수동 제어

// 코드로 Path Position 제어
[SerializeField] private CinemachineVirtualCamera _dollyCam;

public void SetDollyPosition(float position)
{
    var dolly = _dollyCam.GetCinemachineComponent<CinemachineTrackedDolly>();
    dolly.m_PathPosition = position;
}
```

---

## Aim 알고리즘 (카메라 회전)

Look At 타겟 기준으로 카메라 회전을 결정하는 알고리즘.

### 알고리즘 선택 기준

| 알고리즘 | 용도 |
|---|---|
| `Composer` | 일반 조준. Dead Zone/Soft Zone으로 구도 설정 |
| `Group Composer` | 다중 타겟 조준. CinemachineTargetGroup 사용 |
| `POV` | FPS/마우스 룩. 사용자 입력으로 회전 |
| `Hard Look At` | 타겟 정중앙 고정 |
| `Same As Follow Target` | Follow 타겟의 회전 적용 |
| `Do Nothing` | 회전 변경 없음 |

### Composer

```csharp
// Inspector에서 설정
// Aim → Composer
// Tracked Object Offset: (0, 1.5, 0) — 타겟 머리 위
// Screen X/Y: 0.5 (화면 중앙)
// Dead Zone Width/Height: 0.1
// Soft Zone Width/Height: 0.8
// Horizontal/Vertical Damping: 1
```

**화면 영역 구성:**

| 영역 | 동작 |
|---|---|
| **Dead Zone** (투명) | 타겟이 이 안에 있으면 카메라 회전 없음 |
| **Soft Zone** (파란색) | 타겟이 이 안에 들어오면 부드럽게 Dead Zone으로 복귀 |
| **Hard Limit** (빨간색) | 타겟이 이 밖으로 나가지 않도록 즉시 회전 |

### POV (1인칭 / 마우스 룩)

```csharp
// Inspector에서 설정
// Aim → POV
// Vertical Axis > Input Axis Name: "Mouse Y"
// Horizontal Axis > Input Axis Name: "Mouse X"
// Vertical Axis > Value Range: -70, 70
// Recentering > Enabled: false (FPS에서는 보통 비활성화)
```

### Group Composer (다중 타겟)

```csharp
// 1. 빈 GameObject에 CinemachineTargetGroup 추가
// 2. Targets 리스트에 추적할 오브젝트 추가 (Weight, Radius 설정)
// 3. Virtual Camera의 Look At에 TargetGroup 할당
// 4. Aim → Group Composer 선택
// → FOV 또는 카메라 거리가 자동 조정되어 모든 타겟을 프레임에 유지

[SerializeField] private CinemachineTargetGroup _targetGroup;

public void AddTarget(Transform target, float weight = 1f, float radius = 1f)
{
    _targetGroup.AddMember(target, weight, radius);
}

public void RemoveTarget(Transform target)
{
    _targetGroup.RemoveMember(target);
}
```

---

## 감쇠 (Damping)

Body와 Aim 모두에 적용되는 추적 응답 속도 설정.

| 값 | 효과 |
|---|---|
| `0` | 즉각 응답 (딱딱함) |
| `1~3` | 부드러운 추적 (일반적) |
| `5+` | 매우 느린 추적 (무거운 카메라) |

```
일반 규칙: 숫자가 작을수록 빠르게 응답, 클수록 느리게 응답
```

---

## Noise (카메라 흔들림)

### Basic Multi Channel Perlin

```csharp
// Inspector에서 설정
// Noise → Basic Multi Channel Perlin
// Noise Profile: 내장 프리셋 선택 (Handheld_normal_mild 등)
// Amplitude Gain: 1 (흔들림 강도 배수)
// Frequency Gain: 1 (흔들림 빈도 배수)

// 코드로 Noise 강도 제어
[SerializeField] private CinemachineVirtualCamera _cam;

public void SetShakeIntensity(float amplitude, float frequency)
{
    var noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
    noise.m_AmplitudeGain = amplitude;
    noise.m_FrequencyGain = frequency;
}

// 흔들림 시작/중지
public void StartShake(float amplitude = 1f, float frequency = 1f)
{
    var noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
    noise.m_AmplitudeGain = amplitude;
    noise.m_FrequencyGain = frequency;
}

public void StopShake()
{
    var noise = _cam.GetCinemachineComponent<CinemachineBasicMultiChannelPerlin>();
    noise.m_AmplitudeGain = 0f;
}
```

### 노이즈 프로파일 생성

```
프로젝트 창 → Create > Cinemachine > NoiseSettings

Position Noise (위치): X/Y/Z 축별 Frequency(Hz), Amplitude(거리) 설정
Rotation Noise (회전): X/Y/Z 축별 Frequency(Hz), Amplitude(각도) 설정

권장: 회전 노이즈를 먼저 설정하고 위치 노이즈를 추가
- 저주파: 0.1~0.5 Hz (느린 흔들림)
- 중주파: 0.8~1.5 Hz (핸드헬드)
- 고주파: 3~4 Hz (차량/폭발)
```

---

## 블렌딩 (카메라 전환)

카메라 간 전환 시 위치, 회전, FOV 등을 부드럽게 보간한다 (페이드/와이프가 아님).

### 블렌딩 타입

| 스타일 | 특성 | 용도 |
|---|---|---|
| `Cut` | 즉시 전환 | 빠른 전환, 컷씬 |
| `Ease In Out` | S자 곡선 (기본) | 일반 전환 |
| `Ease In` | 들어오는 카메라 서서히 시작 | 드라마틱 진입 |
| `Ease Out` | 나가는 카메라 서서히 종료 | 부드러운 퇴장 |
| `Hard In` | 빠른 시작, 느린 종료 | 강조 |
| `Hard Out` | 느린 시작, 빠른 종료 | 긴장감 |
| `Linear` | 균등 보간 | 기계적 전환 |
| `Custom` | 사용자 정의 커브 | 특수 효과 |

### 커스텀 블렌딩 설정

```
1. Assets > Create > Cinemachine > Blending Profile
2. From/To 카메라 이름 지정 (또는 "**ANY CAMERA**")
3. Style과 Time 설정
4. Brain의 Custom Blends에 할당
```

---

## 특수 카메라 타입

### FreeLook Camera (3인칭 오비탈)

3개 궤도(Top, Middle, Bottom)를 사용하여 타겟 주위를 자유롭게 회전.

```csharp
// 메뉴: Cinemachine > Create FreeLook Camera
// 3개 리그(Orbit) 설정:
//   Top:    Height = 4, Radius = 5
//   Middle: Height = 2, Radius = 8
//   Bottom: Height = 0, Radius = 5

// X Axis: 수평 회전 (마우스/스틱 입력)
// Y Axis: 수직 궤도 전환 (Top ↔ Bottom)
// Spline Curvature: 리그 간 곡선 부드러움 (0~1)
```

### ClearShot Camera

자식 카메라 중 최적의 시야(장애물 없는)를 자동 선택.

```csharp
// 메뉴: Cinemachine > Create ClearShot Camera
// 자식으로 여러 Virtual Camera 추가
// 각 자식에 CinemachineCollider Extension 추가
// → 장애물 분석으로 최고 품질의 샷 자동 선택

// Activate After: 새 카메라 활성화 전 대기 시간 (초)
// Min Duration: 활성화된 카메라 최소 유지 시간 (초)
// Randomize Choice: 동일 품질 시 무작위 선택
```

### State-Driven Camera

Animator 상태에 따라 자식 카메라를 자동 전환.

```csharp
// 메뉴: Cinemachine > Create State-Driven Camera
// Animated Target: Animator가 있는 GameObject 지정
// State 리스트에서 각 애니메이션 상태에 자식 카메라 매핑
//   Idle → IdleCam
//   Run  → RunCam
//   Jump → JumpCam
```

### Blend List Camera

미리 정의된 순서대로 자식 카메라를 자동 전환 (컷씬용).

```csharp
// 메뉴: Cinemachine > Create Blend List Camera
// Instructions 리스트에서 순차적 카메라 전환 정의:
//   Camera 1 → 3초 유지 → Ease In Out → Camera 2
//   Camera 2 → 2초 유지 → Cut → Camera 3
```

### Mixing Camera

최대 8개 자식 카메라를 가중치로 혼합.

```csharp
// 메뉴: Cinemachine > Create Mixing Camera
// 각 자식 카메라의 Weight를 실시간 조절
// 가중치 비율로 위치/회전/FOV를 혼합
// Timeline에서 Weight 애니메이션 가능
```

---

## Extensions

Virtual Camera에 추가 동작을 부여하는 컴포넌트.

### Confiner (영역 제한)

카메라를 지정한 영역 안에 가둔다.

```csharp
// Virtual Camera Inspector → Add Extension → CinemachineConfiner
// Bounding Volume: 3D Collider (Box, Sphere 등)
// Bounding Shape 2D: Collider2D (Polygon, Box 등)
// Confine Screen Edges: 직교 카메라에서 화면 가장자리도 제한
// Damping: 경계 복귀 부드러움
```

### Collider (장애물 회피)

카메라와 타겟 사이의 장애물을 감지하고 카메라를 이동.

```csharp
// Virtual Camera Inspector → Add Extension → CinemachineCollider
// Collide Against: 장애물 레이어 마스크
// Avoid Obstacles: 활성화
// Camera Radius: 장애물로부터 유지 거리
// Damping: 정상 포지션 복귀 속도
// Strategy: Pull Camera Forward / Preserve Camera Height / Preserve Camera Distance
```

### Impulse Listener (충격 반응)

Impulse Source가 발생시킨 진동에 반응하여 카메라를 흔든다.

```csharp
// Virtual Camera Inspector → Add Extension → CinemachineImpulseListener
// Gain: 진동 강도 배수
// Channel Mask: 수신할 채널 선택

// Impulse Source 설정 (별도 GameObject)
[SerializeField] private CinemachineImpulseSource _impulseSource;

public void TriggerShake(Vector3 velocity)
{
    _impulseSource.GenerateImpulse(velocity);
}
```

### Post Processing

카메라별 포스트 프로세싱 프로파일 적용.

```csharp
// 사전 설정: Post Processing V2 패키지 설치
// Cinemachine > Import Post Processing V2 Adaptor Asset Package
// Unity Camera에 Post-Process Layer 추가
// Virtual Camera → Add Extension → CinemachinePostProcessing
// Profile: 포스트 프로세싱 프로파일 에셋
// Focus Tracks Target: 카메라-타겟 거리로 자동 초점
```

---

## Impulse 시스템 (충격 카메라 흔들림)

게임 이벤트에 의한 카메라 흔들림. Noise와 달리 일회성 충격 효과.

### 구성

```
Impulse Source (발신) → 씬 공간에 진동 신호 방출
Impulse Listener (수신) → Virtual Camera Extension으로 진동 수신
```

### Impulse Source 설정

```csharp
// CinemachineImpulseSource: 코드에서 수동 트리거
// CinemachineCollisionImpulseSource: 충돌/트리거 자동 트리거

[SerializeField] private CinemachineImpulseSource _impulseSource;

// 기본 충격
public void OnHit()
{
    _impulseSource.GenerateImpulse();
}

// 방향성 충격
public void OnExplosion(Vector3 direction, float force)
{
    _impulseSource.GenerateImpulse(direction.normalized * force);
}
```

### Time Envelope

| 구간 | 설명 |
|---|---|
| `Attack` | 최대 강도까지의 페이드인 |
| `Sustain Time` | 최대 강도 유지 시간 |
| `Decay` | 0으로의 페이드아웃 |

### Spatial Range

| 프로퍼티 | 설명 |
|---|---|
| `Impact Radius` | 최대 강도 유지 거리 |
| `Dissipation Distance` | 신호가 약해지는 거리 |
| `Dissipation Mode` | Exponential / Soft / Linear Decay |

---

## Path (경로)

Tracked Dolly Body 알고리즘에서 사용하는 카메라 이동 경로.

### Cinemachine Path

| 프로퍼티 | 설명 |
|---|---|
| `Resolution` | 웨이포인트당 세분화 수 |
| `Looped` | 루프 경로 여부 |
| `Waypoints > Position` | 경로 로컬 공간 위치 |
| `Waypoints > Tangent` | 커브 탄젠트 방향 |
| `Waypoints > Roll` | 해당 지점의 롤 회전 |

### Cinemachine Smooth Path

베지어 커브로 보간되는 부드러운 경로. Waypoint에 Tangent 대신 자동 보간.

---

## Target Group (다중 타겟)

여러 오브젝트를 하나의 타겟으로 묶어 Group Composer와 함께 사용.

```csharp
// 빈 GameObject → Add Component → CinemachineTargetGroup

// Position Mode:
//   Group Center: 바운딩 박스 중심
//   Group Average: 가중 평균 위치

// Rotation Mode:
//   Manual: Transform 회전 직접 사용
//   Group Average: 멤버 방향의 가중 평균

// 각 타겟: Weight (가중치), Radius (바운딩 반경)
```

---

## Timeline 연동

Timeline에서 Cinemachine Track으로 카메라를 시퀀스 제어.

```
1. Timeline 창 → Cinemachine Brain이 있는 카메라를 드래그 → Cinemachine Track 생성
2. Cinemachine Track에 Shot Clip 추가
3. 각 Shot Clip에 Virtual Camera 할당
4. 클립 나란히 배치 → Cut / 겹쳐 배치 → Blending
5. Timeline 재생 중에는 Brain의 우선순위 시스템을 오버라이드
```

> 단순한 샷 시퀀스에는 Timeline 대신 Blend List Camera 사용을 권장.

---

## 장르별 추천 구성

| 장르 | Body | Aim | 비고 |
|---|---|---|---|
| 3인칭 액션 | Transposer | Composer | 기본 구성 |
| 3인칭 오비탈 | FreeLook Camera (3 Rig) | Composer | 사용자 회전 |
| FPS | Hard Lock to Target | POV | 1인칭 |
| 2D 플랫포머 | Framing Transposer | Do Nothing | + Confiner |
| 탑뷰/RTS | Transposer (World Space) | Hard Look At | 고정 앵글 |
| 레일 카메라 | Tracked Dolly | Composer | Path 필요 |
| 컷씬 | 여러 카메라 | 혼합 | Timeline 또는 Blend List |
| 보스전 | Group Composer (Aim) | Framing Transposer | TargetGroup |

---

## 자주 하는 실수 / 금지 패턴

| 하지 말 것 | 대신 할 것 |
|---|---|
| Unity Camera의 Transform을 직접 조작 | Virtual Camera 전환 또는 프로퍼티 변경 |
| 하나의 Virtual Camera에서 런타임에 Body/Aim 변경 | 별도 Virtual Camera 만들고 전환 |
| Priority를 같은 값으로 설정 | 명확한 우선순위 계층 유지 |
| Damping 0으로 설정 (딱딱함) | 최소 0.5~1 부여 |
| Noise의 AmplitudeGain을 항상 높게 유지 | 이벤트 시에만 활성화 후 0으로 복구 |
| Confiner 없이 2D 카메라 사용 | PolygonCollider2D + Confiner 설정 |
| ClearShot 자식에 Collider Extension 미추가 | 반드시 Collider 추가해야 품질 평가 작동 |
| 코드로 Brain을 직접 생성 | 에디터에서 추가하거나 Cinemachine 메뉴 사용 |

---

## 참조 파일 가이드

복잡한 케이스는 아래 기준으로 참조 파일을 읽는다.

| 이런 작업이 필요할 때 | 참조 파일 |
|---|---|
| Impulse 시스템 상세 설정, 채널 필터링, 충돌 임펄스 | `references/impulse-system.md` |
| 2D 카메라 구성, Confiner 상세, 데드존/소프트존 튜닝 | `references/camera-patterns.md` |
