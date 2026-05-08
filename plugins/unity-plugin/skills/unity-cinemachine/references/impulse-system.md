# Impulse 시스템 상세

## Impulse Source 타입

### CinemachineImpulseSource

코드에서 수동으로 트리거하는 임펄스 소스.

```csharp
using Cinemachine;
using UnityEngine;

public class ExplosionEffect : MonoBehaviour
{
    [SerializeField] private CinemachineImpulseSource _impulseSource;

    public void Explode(float force)
    {
        // 기본 방향으로 임펄스 생성
        _impulseSource.GenerateImpulse(force);
    }

    public void DirectionalExplosion(Vector3 direction, float force)
    {
        // 방향성 임펄스
        _impulseSource.GenerateImpulse(direction.normalized * force);
    }

    public void CustomVelocityImpulse()
    {
        // 정확한 Velocity 벡터로 임펄스 생성
        _impulseSource.GenerateImpulse(new Vector3(0.2f, -0.5f, 0f));
    }
}
```

### CinemachineCollisionImpulseSource

충돌 또는 트리거에 자동 반응하는 임펄스 소스.

```csharp
// Inspector에서 설정:
// - How To Generate The Impulse: Collision / Trigger
// - Layer Mask: 반응할 레이어
// - Ignore Tag: 무시할 태그
// - Use Impact Direction: 충돌 방향으로 임펄스 방향 설정
```

---

## Raw Signal (진동 신호)

임펄스의 진동 패턴을 정의하는 에셋.

### 노이즈 프로파일 기반

6D 노이즈 (Position X/Y/Z + Rotation X/Y/Z)를 사용:

| 축 | Frequency (Hz) | Amplitude | 효과 |
|---|---|---|---|
| Position Y | 10-15 Hz | 0.1-0.3 | 수직 충격 |
| Rotation Z | 8-12 Hz | 1-3° | 회전 흔들림 |
| Position X/Z | 5-8 Hz | 0.05-0.15 | 수평 진동 |

### 고정 신호 (Fixed Signal)

미리 정의된 3D 커브로 정밀한 진동 패턴 제어.

```
프로젝트 창 → Create > Cinemachine > Fixed Signal
→ X/Y/Z 축별 AnimationCurve 편집
```

### 커스텀 신호

```csharp
// CinemachineSignalSource를 상속하여 커스텀 진동 패턴 구현
// 자동으로 Inspector의 Raw Signal 드롭다운에 표시됨
```

---

## Time Envelope 상세

임펄스의 시간 경과에 따른 강도 변화.

```
┌─────────────────────────────────────────┐
│         ╱‾‾‾‾‾‾‾‾‾‾‾╲                   │
│        ╱  Sustain     ╲                  │
│       ╱                ╲                 │
│      ╱                  ╲                │
│     ╱                    ╲               │
│────╱  Attack   │  Decay   ╲────          │
│                                          │
└─────────────────────────────────────────┘
```

| 프로퍼티 | 권장값 | 용도 |
|---|---|---|
| Attack | 0 ~ 0.1s | 충격 (즉시) |
| Attack | 0.3 ~ 0.5s | 지진 (서서히) |
| Sustain | 0.1 ~ 0.3s | 짧은 충격 |
| Sustain | 1 ~ 3s | 지속 흔들림 |
| Decay | 0.2 ~ 0.5s | 일반 감쇠 |
| Decay | 1 ~ 2s | 여운 있는 감쇠 |

---

## Spatial Range 상세

임펄스의 공간적 영향 범위.

```
       ┌─ Impact Radius ─┐
       │                  │
Source ●══════════════════╪─────────────────╪ (강도 0)
       │  최대 강도 유지  │  Dissipation    │
       │                  │  Distance       │
       └──────────────────┴─────────────────┘
```

### Dissipation Mode

| 모드 | 특성 | 용도 |
|---|---|---|
| `Exponential Decay` | 빠르게 감쇠 후 긴 꼬리 | 폭발 (기본) |
| `Soft Decay` | 부드러운 감쇠 커브 | 자연 현상 |
| `Linear Decay` | 균등 감쇠 | 기계적 진동 |

---

## 채널 필터링

특정 리스너만 특정 소스에 반응하도록 필터링.

### 설정 방법

```csharp
// 1. 채널 정의 (최대 31개)
//    CinemachineImpulseChannels.asset 스크립트에서 채널 이름 편집

// 2. Impulse Source → Impulse Channel: 브로드캐스트할 채널 선택
// 3. Impulse Listener → Channel Mask: 수신할 채널 선택
```

### 사용 시나리오

| 시나리오 | 소스 채널 | 리스너 채널 |
|---|---|---|
| 전투 카메라만 흔들림 | "Combat" | "Combat" |
| UI 카메라 흔들림 방지 | "Gameplay" | UI 카메라는 "Gameplay" 제외 |
| 전체 카메라 흔들림 | 기본 (All) | 기본 (All) |

---

## 충돌 임펄스 소스 프로퍼티

### Layer Mask
선택된 레이어의 오브젝트만 임펄스를 트리거한다.

### Ignore Tag
지정된 태그의 오브젝트는 임펄스를 트리거하지 않는다.

### Use Impact Direction
활성화 시 충돌 방향을 임펄스 방향으로 사용한다.

### Direction Mode

| 모드 | 동작 |
|---|---|
| `Fixed` | 항상 고정 방향 |
| `Rotate Towards Source` | 임펄스 소스 방향으로 회전 |

---

## 실전 패턴

### 피격 흔들림

```csharp
[SerializeField] private CinemachineImpulseSource _hitImpulse;

public void OnDamaged(float damage, Vector3 hitDirection)
{
    float intensity = Mathf.Clamp01(damage / 100f);
    _hitImpulse.GenerateImpulse(hitDirection * intensity);
}
```

### 착지 충격

```csharp
[SerializeField] private CinemachineImpulseSource _landImpulse;

public void OnLand(float fallHeight)
{
    if (fallHeight > 3f)
    {
        float intensity = Mathf.Clamp01((fallHeight - 3f) / 10f);
        _landImpulse.GenerateImpulse(Vector3.down * intensity);
    }
}
```

### 폭발 범위 기반

```csharp
[SerializeField] private CinemachineImpulseSource _explosionImpulse;

public void Explode(Vector3 position, float radius, float force)
{
    // Spatial Range가 설정되어 있으면 거리에 따라 자동 감쇠
    transform.position = position;
    _explosionImpulse.GenerateImpulse(Vector3.up * force);
}
```
