# DOTween 컴포넌트별 숏컷 레퍼런스

## Transform

### 이동
| 메서드 | 설명 |
|---|---|
| `DOMove(Vector3, duration, snapping)` | 월드 좌표 이동 |
| `DOMoveX/Y/Z(float, duration, snapping)` | 단일 축 이동 |
| `DOLocalMove(Vector3, duration, snapping)` | 로컬 좌표 이동 |
| `DOLocalMoveX/Y/Z(float, duration, snapping)` | 로컬 단일 축 |
| `DOJump(endValue, jumpPower, numJumps, duration, snapping)` | 점프 이동 |
| `DOLocalJump(...)` | 로컬 점프 |

### 회전
| 메서드 | 설명 |
|---|---|
| `DORotate(Vector3, duration, RotateMode)` | 오일러 회전 |
| `DORotateQuaternion(Quaternion, duration)` | 쿼터니언 회전 |
| `DOLocalRotate(Vector3, duration, RotateMode)` | 로컬 회전 |
| `DOLocalRotateQuaternion(Quaternion, duration)` | 로컬 쿼터니언 |
| `DOLookAt(towards, duration, AxisConstraint, up)` | 대상 바라보기 |

### 스케일
| 메서드 | 설명 |
|---|---|
| `DOScale(float/Vector3, duration)` | 스케일 |
| `DOScaleX/Y/Z(float, duration)` | 단일 축 스케일 |

### Punch / Shake
| 메서드 | 설명 |
|---|---|
| `DOPunchPosition(punch, duration, vibrato, elasticity, snapping)` | 위치 펀치 |
| `DOPunchRotation(punch, duration, vibrato, elasticity)` | 회전 펀치 |
| `DOPunchScale(punch, duration, vibrato, elasticity)` | 스케일 펀치 |
| `DOShakePosition(duration, strength, vibrato, randomness, snapping, fadeOut)` | 위치 흔들기 |
| `DOShakeRotation(duration, strength, vibrato, randomness, fadeOut)` | 회전 흔들기 |
| `DOShakeScale(duration, strength, vibrato, randomness, fadeOut)` | 스케일 흔들기 |

### Path
| 메서드 | 설명 |
|---|---|
| `DOPath(waypoints, duration, PathType, PathMode, resolution, gizmoColor)` | 경로 이동 |
| `DOLocalPath(...)` | 로컬 경로 이동 |

PathType: `Linear`, `CatmullRom` (곡선), `CubicBezier` (베지어)

### Blendable (중첩 가능)
| 메서드 | 설명 |
|---|---|
| `DOBlendableMoveBy(Vector3, duration, snapping)` | 블렌딩 이동 |
| `DOBlendableLocalMoveBy(Vector3, duration, snapping)` | 블렌딩 로컬 이동 |
| `DOBlendableRotateBy(Vector3, duration, RotateMode)` | 블렌딩 회전 |
| `DOBlendableLocalRotateBy(Vector3, duration, RotateMode)` | 블렌딩 로컬 회전 |
| `DOBlendableScaleBy(Vector3, duration)` | 블렌딩 스케일 |

> Blendable 트윈은 여러 개가 동시에 동작하면서 값이 합산된다.
> 일반 트윈은 마지막 것만 적용되지만, Blendable은 모두 합쳐진다.

---

## Material

| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 메인 색상 |
| `DOColor(Color, property, duration)` | 특정 프로퍼티 색상 |
| `DOFade(float, duration)` | 메인 알파 |
| `DOFade(float, property, duration)` | 특정 프로퍼티 알파 |
| `DOFloat(float, property, duration)` | float 프로퍼티 |
| `DOVector(Vector4, property, duration)` | Vector4 프로퍼티 |
| `DOOffset(Vector2, duration)` | 텍스처 오프셋 |
| `DOTiling(Vector2, duration)` | 텍스처 타일링 |
| `DOGradientColor(Gradient, duration)` | 그라디언트 색상 |
| `DOBlendableColor(Color, duration)` | 블렌딩 색상 |

---

## SpriteRenderer

| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 색상 |
| `DOFade(float, duration)` | 알파 |
| `DOGradientColor(Gradient, duration)` | 그라디언트 |
| `DOBlendableColor(Color, duration)` | 블렌딩 색상 |

---

## Camera

| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 배경색 |
| `DOFieldOfView(float, duration)` | FOV |
| `DOOrthoSize(float, duration)` | 직교 크기 |
| `DONearClipPlane(float, duration)` | 니어 클립 |
| `DOFarClipPlane(float, duration)` | 파 클립 |
| `DOAspect(float, duration)` | 종횡비 |
| `DORect(Rect, duration)` | 뷰포트 |
| `DOShakePosition(duration, strength, vibrato, randomness, fadeOut)` | 위치 흔들기 |
| `DOShakeRotation(duration, strength, vibrato, randomness, fadeOut)` | 회전 흔들기 |

---

## Light

| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 색상 |
| `DOIntensity(float, duration)` | 강도 |
| `DOShadowStrength(float, duration)` | 그림자 강도 |
| `DOBlendableColor(Color, duration)` | 블렌딩 색상 |

---

## AudioSource

| 메서드 | 설명 |
|---|---|
| `DOFade(float, duration)` | 볼륨 |
| `DOPitch(float, duration)` | 피치 |

---

## Rigidbody / Rigidbody2D

| 메서드 | 설명 |
|---|---|
| `DOMove(Vector3/Vector2, duration, snapping)` | MovePosition 사용 |
| `DOMoveX/Y/Z(float, duration, snapping)` | 단일 축 |
| `DORotate(Vector3/float, duration, RotateMode)` | MoveRotation 사용 |
| `DOJump(endValue, jumpPower, numJumps, duration, snapping)` | 점프 |
| `DOPath(waypoints, duration, PathType, PathMode)` | 경로 |

---

## UI (uGUI)

### CanvasGroup
| 메서드 | 설명 |
|---|---|
| `DOFade(float, duration)` | 알파 |

### Image
| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 색상 |
| `DOFade(float, duration)` | 알파 |
| `DOFillAmount(float, duration)` | 채움량 (0~1) |
| `DOGradientColor(Gradient, duration)` | 그라디언트 |
| `DOBlendableColor(Color, duration)` | 블렌딩 색상 |

### RectTransform
| 메서드 | 설명 |
|---|---|
| `DOAnchorPos(Vector2, duration, snapping)` | anchoredPosition |
| `DOAnchorPosX/Y(float, duration, snapping)` | 단일 축 |
| `DOAnchorPos3D(Vector3, duration, snapping)` | 3D 앵커 |
| `DOSizeDelta(Vector2, duration, snapping)` | sizeDelta |
| `DOPivot(Vector2, duration)` | 피벗 |
| `DOJumpAnchorPos(endValue, jumpPower, numJumps, duration, snapping)` | 앵커 점프 |
| `DOPunchAnchorPos(punch, duration, vibrato, elasticity, snapping)` | 앵커 펀치 |
| `DOShakeAnchorPos(duration, strength, vibrato, randomness, snapping, fadeOut)` | 앵커 흔들기 |

### Text
| 메서드 | 설명 |
|---|---|
| `DOColor(Color, duration)` | 색상 |
| `DOFade(float, duration)` | 알파 |
| `DOText(string, duration, richText, ScrambleMode, scrambleChars)` | 타이핑 효과 |
| `DOBlendableColor(Color, duration)` | 블렌딩 색상 |

### Slider
| 메서드 | 설명 |
|---|---|
| `DOValue(float, duration, snapping)` | 값 |

### ScrollRect
| 메서드 | 설명 |
|---|---|
| `DONormalizedPos(Vector2, duration, snapping)` | 정규화 위치 |
| `DOHorizontalNormalizedPos(float, duration, snapping)` | 수평 |
| `DOVerticalPos(float, duration, snapping)` | 수직 |

---

## 범용 (DOTween.To)

| 메서드 | 설명 |
|---|---|
| `DOTween.To(getter, setter, endValue, duration)` | 모든 타입 트윈 |
| `DOTween.ToAlpha(getter, setter, endAlpha, duration)` | Color 알파 |
| `DOTween.ToAxis(getter, setter, endValue, duration, axis)` | Vector3 단일 축 |
| `DOTween.ToArray(getter, setter, endValues[], durations[])` | 다단계 트윈 |
| `DOTween.Punch(getter, setter, direction, duration, vibrato, elasticity)` | 범용 펀치 |
| `DOTween.Shake(getter, setter, duration, strength, vibrato, randomness)` | 범용 흔들기 |

---

## Ease 전체 목록

### 표준
Linear, InSine, OutSine, InOutSine, InQuad, OutQuad, InOutQuad, InCubic, OutCubic, InOutCubic, InQuart, OutQuart, InOutQuart, InQuint, OutQuint, InOutQuint, InExpo, OutExpo, InOutExpo, InCirc, OutCirc, InOutCirc, InElastic, OutElastic, InOutElastic, InBack, OutBack, InOutBack, InBounce, OutBounce, InOutBounce

### 특수
Flash, InFlash, OutFlash, InOutFlash

### 커스텀
`AnimationCurve` 또는 `EaseFunction` 델리게이트
