---
paths:
  - "**/*.cs"
---

# Unity C# 코딩 표준 가이드

> **적용 범위**: 프로젝트 내 모든 `*.cs` 파일  
> **버전**: Unity 6 / C# 9+

---

## 목차

1. [파일 & 폴더 구조](#1-파일--폴더-구조)
2. [네임스페이스](#2-네임스페이스)
3. [명명 규칙](#3-명명-규칙)
4. [포매팅](#4-포매팅)
5. [클래스 멤버 선언 순서](#5-클래스-멤버-선언-순서)
6. [주석 & 문서화](#6-주석--문서화)
7. [Unity 전용 규칙](#7-unity-전용-규칙)
8. [성능 가이드라인](#8-성능-가이드라인)
9. [금지 사항](#9-금지-사항)
10. [예제 파일](#10-예제-파일)

---

## 1. 파일 & 폴더 구조

```
Assets/
├── _Project/               ← 프로젝트 전용 루트 (밑줄로 최상단 고정)
│   ├── Scripts/
│   │   ├── Runtime/        ← MonoBehaviour, ScriptableObject 등
│   │   ├── Editor/         ← Editor 전용 스크립트
│   │   └── Tests/          ← PlayMode / EditMode 테스트
│   ├── Prefabs/
│   ├── Scenes/
│   └── ...
```

- **파일명 == 클래스명** (대소문자 완전 일치)  
- 파일 하나에 **public 클래스 하나** (private 헬퍼 클래스는 같은 파일 허용)  
- 인코딩: **UTF-8 without BOM**

---

## 2. 네임스페이스

```csharp
// ✅ DO: 회사.프로젝트.기능 계층 구조
namespace AcmeCorp.ProjectName.Gameplay
{
    public class PlayerController : MonoBehaviour { }
}

// ❌ DON'T: 네임스페이스 없이 전역 선언
public class PlayerController : MonoBehaviour { }

// ❌ DON'T: using static 남용
using static UnityEngine.Mathf;
```

- `using` 지시문은 **네임스페이스 블록 외부** 최상단에 배치  
- `System.*` → `UnityEngine.*` → `UnityEditor.*` → 서드파티 → 내부 순 정렬  
- 불필요한 `using` 제거 (IDE 경고 활성화)

---

## 3. 명명 규칙

### 3-1. 요약표

| 대상 | 스타일 | 예시 |
|------|--------|------|
| 클래스 / 구조체 / 열거형 / 델리게이트 | `PascalCase` | `PlayerController` |
| 인터페이스 | `I` + `PascalCase` | `IDamageable` |
| 메서드 / 프로퍼티 / 이벤트 | `PascalCase` | `TakeDamage()`, `IsAlive` |
| public 필드 | `PascalCase` | `MaxHealth` |
| private / protected 필드 | `_camelCase` | `_currentHealth` |
| private static 필드 | `s_camelCase` | `s_instance` |
| 상수 (`const`) | `k_UPPER_CASE` | `k_MAX_SPEED` |
| 로컬 변수 / 파라미터 | `camelCase` | `spawnPosition` |
| 로컬 함수 | `PascalCase` | `ComputeDamage()` |
| 열거형 값 | `PascalCase` | `GameState.Playing` |

### 3-2. 세부 규칙

```csharp
// ── 인터페이스 ──────────────────────────────────────────
public interface IDamageable
{
    void TakeDamage(float amount);
}

// ── 클래스 ──────────────────────────────────────────────
public class EnemyController : MonoBehaviour, IDamageable
{
    // ── 상수 ─────────────────────────────────────────────
    private const float k_GRAVITY_SCALE = 9.81f;
    private const int   k_MAX_POOL_SIZE = 64;

    // ── static 필드 ──────────────────────────────────────
    private static int s_spawnCount;

    // ── SerializeField (private 필드) ────────────────────
    [SerializeField] private float _moveSpeed = 5f;
    [SerializeField] private int   _maxHealth = 100;

    // ── private 필드 ─────────────────────────────────────
    private int   _currentHealth;
    private bool  _isDead;

    // ── 프로퍼티 (PascalCase) ────────────────────────────
    public bool IsAlive => _currentHealth > 0;

    // ── 이벤트 ───────────────────────────────────────────
    public event Action<float> OnDamageTaken;
    public event Action        OnDeath;

    // ── 메서드 ───────────────────────────────────────────
    public void TakeDamage(float amount)
    {
        if (_isDead) { return; }

        _currentHealth -= (int)amount;
        OnDamageTaken?.Invoke(amount);

        if (_currentHealth <= 0)
        {
            Die();
        }
    }

    private void Die()
    {
        _isDead = true;
        OnDeath?.Invoke();
    }
}
```

### 3-3. 금지된 명명

```csharp
// ❌ 헝가리안 표기법
private int iCount;
private string strName;

// ❌ 축약어 (일반적으로 알려진 것 제외: ID, UI, AI, HP, MP …)
private float spd;
private GameObject go;

// ❌ 단일 문자 변수 (루프 카운터 i, j, k 는 허용)
private float x;

// ❌ 이중 밑줄
private int __value;

// ❌ 대문자 약어 (2자 초과 시 PascalCase 적용)
// string URL → string Url
// int HTTPCode → int HttpCode
```

---

## 4. 포매팅

### 4-1. 중괄호 — Allman 스타일 (필수)

```csharp
// ✅ DO: 항상 새 줄에 여는 중괄호
if (condition)
{
    DoSomething();
}
else
{
    DoOther();
}

// ❌ DON'T: K&R / 한 줄 스타일
if (condition) { DoSomething(); }
if (condition) DoSomething();  // 중괄호 생략 금지
```

### 4-2. 들여쓰기 & 줄 길이

- 들여쓰기: **공백 4칸** (탭 금지)  
- 최대 줄 길이: **120자**  
- 긴 파라미터 목록은 줄 바꿈 후 한 단계 들여쓰기

```csharp
// ✅ 긴 파라미터 줄 바꿈
public void SpawnEnemy(
    Vector3 position,
    Quaternion rotation,
    EnemyType type,
    int level)
{
    // ...
}
```

### 4-3. 공백 규칙

```csharp
// ✅ 연산자 주변 공백
int total = a + b * c;
bool isValid = count > 0 && count < k_MAX_POOL_SIZE;

// ✅ 콤마 뒤 공백
DoSomething(arg1, arg2, arg3);

// ✅ 제어문 키워드 뒤 공백
if (condition) { }
for (int i = 0; i < 10; i++) { }

// ❌ 괄호 안 공백
if ( condition ) { }        // 금지
DoSomething( arg1, arg2 ); // 금지
```

### 4-4. 빈 줄

```csharp
public class Example : MonoBehaviour
{
    // 필드 그룹 사이: 빈 줄 1개
    [SerializeField] private float _speed;
    [SerializeField] private int   _health;

    private bool _isActive;

    // 메서드 사이: 빈 줄 1개
    private void Awake()
    {
        Initialize();
    }

    private void Initialize()
    {
        _isActive = true;
    }
}
```

---

## 5. 클래스 멤버 선언 순서

다음 순서를 **반드시** 준수합니다.

```
1. 상수 (const)
2. static 필드
3. SerializeField (직렬화 필드)
4. 일반 private / protected 필드
5. 프로퍼티
6. 이벤트 & 델리게이트
7. ─── Unity 생명주기 ──────────────────
   Awake → OnEnable → Start → FixedUpdate
   → Update → LateUpdate → OnDisable → OnDestroy
8. public 메서드
9. protected 메서드
10. private 메서드
11. Editor-only 블록 (#if UNITY_EDITOR)
```

```csharp
public class OrderExample : MonoBehaviour
{
    // 1. 상수
    private const int k_MAX_COUNT = 10;

    // 2. static 필드
    private static int s_totalCreated;

    // 3. SerializeField
    [SerializeField] private float _speed = 5f;

    // 4. private 필드
    private Rigidbody _rigidbody;
    private bool      _isGrounded;

    // 5. 프로퍼티
    public float Speed => _speed;

    // 6. 이벤트
    public event Action OnJump;

    // 7. Unity 생명주기
    private void Awake()      { CacheComponents(); }
    private void Start()      { Initialize(); }
    private void Update()     { HandleInput(); }
    private void FixedUpdate(){ ApplyMovement(); }
    private void OnDestroy()  { Cleanup(); }

    // 8. public 메서드
    public void Jump() { OnJump?.Invoke(); }

    // 9. private 메서드
    private void CacheComponents()  { _rigidbody = GetComponent<Rigidbody>(); }
    private void Initialize()       { s_totalCreated++; }
    private void HandleInput()      { }
    private void ApplyMovement()    { }
    private void Cleanup()          { s_totalCreated--; }

#if UNITY_EDITOR
    private void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.cyan;
        Gizmos.DrawWireSphere(transform.position, 1f);
    }
#endif
}
```

---

## 6. 주석 & 문서화

### 6-1. XML 문서 주석 (public API 필수)

```csharp
/// <summary>
/// 적에게 피해를 입히고 사망 처리를 수행합니다.
/// </summary>
/// <param name="amount">입힐 피해량 (0 이상).</param>
/// <returns>처리 후 남은 체력.</returns>
/// <exception cref="ArgumentOutOfRangeException">amount가 음수인 경우.</exception>
public int TakeDamage(float amount) { ... }
```

### 6-2. 인라인 주석

```csharp
// ✅ DO: '왜(Why)'를 설명
// 물리 충돌 직후 속도를 초기화해야 다음 프레임 오차를 방지할 수 있다.
_rigidbody.velocity = Vector3.zero;

// ❌ DON'T: '무엇(What)'의 반복 — 코드 자체가 설명해야 함
// velocity를 Vector3.zero로 설정한다.
_rigidbody.velocity = Vector3.zero;
```

### 6-3. TODO / FIXME / HACK / NOTE

```csharp
// TODO(홍길동): 풀링 시스템으로 교체 필요 — 2025-Q3
// FIXME: iOS 14 이하에서 패널티 타이머 버그 발생
// HACK: 임시 우회. 엔진 업데이트 후 제거 예정
// NOTE: 이 값은 서버 응답 시간 기준으로 조정됨
```

---

## 7. Unity 전용 규칙

### 7-1. SerializeField vs public 필드

```csharp
// ✅ DO: private + [SerializeField] — 캡슐화 유지
[SerializeField] private GameObject _bulletPrefab;

// ❌ DON'T: Inspector 노출만을 위한 public 필드
public GameObject bulletPrefab;
```

### 7-2. GetComponent 캐싱

```csharp
// ✅ Awake에서 캐싱
private void Awake()
{
    _animator  = GetComponent<Animator>();
    _rigidbody = GetComponent<Rigidbody>();
}

// ❌ Update/LateUpdate 내 매 프레임 호출
private void Update()
{
    GetComponent<Animator>().Play("Run"); // 절대 금지
}
```

### 7-3. CompareTag 사용

```csharp
// ✅ DO
if (other.CompareTag("Enemy")) { ... }

// ❌ DON'T — 문자열 할당 발생
if (other.tag == "Enemy") { ... }
```

### 7-4. null 비교

```csharp
// ✅ Unity Object: == null 비교 (UnityEngine.Object 오버로드 사용)
if (_target == null) { ... }

// ✅ 순수 C# 객체: is null / is not null 패턴
if (_config is null) { ... }

// ❌ Object.ReferenceEquals — Unity Object에 부적합
```

### 7-5. Update 최적화

```csharp
// ✅ 가능하면 이벤트 / 코루틴으로 대체
// ✅ 불필요한 Update는 enabled = false 로 비활성화
// ✅ 반복 문자열 비교는 캐시 처리

private int _animRunHash;

private void Awake()
{
    _animRunHash = Animator.StringToHash("Run"); // 해시 캐싱
}

private void Update()
{
    _animator.SetBool(_animRunHash, IsMoving);
}
```

### 7-6. Coroutine 규칙

```csharp
// ✅ WaitForSeconds 캐싱 (GC 감소)
private static readonly WaitForSeconds s_waitOneSecond = new WaitForSeconds(1f);

private IEnumerator SpawnRoutine()
{
    while (true)
    {
        SpawnEnemy();
        yield return s_waitOneSecond;
    }
}

// ✅ 코루틴 참조 저장 후 명시적 Stop
private Coroutine _spawnCoroutine;

private void OnEnable()  => _spawnCoroutine = StartCoroutine(SpawnRoutine());
private void OnDisable() => StopCoroutine(_spawnCoroutine);
```

---

## 8. 성능 가이드라인

| 항목 | ❌ 금지 | ✅ 권장 |
|------|--------|--------|
| 오브젝트 생성 | `Instantiate` 반복 | **Object Pool** |
| 문자열 결합 루프 | `+` 연산자 | `StringBuilder` |
| LINQ (핫 패스) | `Where`, `Select` 매 프레임 | 루프 직접 작성 |
| Find 계열 | `Find`, `FindObjectOfType` | `Awake` 캐싱 또는 DI |
| Boxing | `int` → `object` 암시적 변환 | 제네릭 / 구조체 활용 |
| Camera.main | 매 프레임 접근 | `Awake`에서 캐싱 |

---

## 9. 금지 사항

```csharp
// ❌ 매직 넘버 — 반드시 상수/열거형으로 교체
if (lives == 3) { ... }              // 금지
if (lives == k_MAX_LIVES) { ... }    // 허용

// ❌ 빈 catch
try { RiskyOperation(); }
catch (Exception) { }                // 반드시 로그 또는 처리 추가

// ❌ GameObject.Find / FindObjectOfType (런타임)
var player = GameObject.Find("Player");  // 성능 위험

// ❌ SendMessage
gameObject.SendMessage("TakeDamage");    // 타입 안전성 없음

// ❌ public 필드로 Inspector 노출
public int health;                       // [SerializeField] private 사용

// ❌ Awake/Start 외부에서 new MonoBehaviour
var enemy = new EnemyController();      // Instantiate 사용
```

---

## 10. 예제 파일

`PlayerHealth.cs` — 모든 규칙 적용 예시

```csharp
using System;
using UnityEngine;

namespace AcmeCorp.ProjectName.Gameplay
{
    /// <summary>
    /// 플레이어의 체력을 관리하고 피해/회복/사망 이벤트를 처리합니다.
    /// </summary>
    public sealed class PlayerHealth : MonoBehaviour, IDamageable
    {
        // ── 상수 ──────────────────────────────────────────
        private const float k_REGEN_INTERVAL = 1f;

        // ── SerializeField ────────────────────────────────
        [SerializeField] private int   _maxHealth  = 100;
        [SerializeField] private float _regenRate  = 5f;
        [SerializeField] private bool  _autoRegen  = true;

        // ── private 필드 ──────────────────────────────────
        private int       _currentHealth;
        private Coroutine _regenCoroutine;

        // ── WaitForSeconds 캐시 ───────────────────────────
        private static readonly WaitForSeconds s_regenWait =
            new WaitForSeconds(k_REGEN_INTERVAL);

        // ── 프로퍼티 ──────────────────────────────────────
        public int   CurrentHealth => _currentHealth;
        public float HealthPercent => (float)_currentHealth / _maxHealth;
        public bool  IsAlive       => _currentHealth > 0;

        // ── 이벤트 ────────────────────────────────────────
        /// <summary>피해를 받을 때 발생합니다. 파라미터: 받은 피해량.</summary>
        public event Action<float> OnDamageTaken;

        /// <summary>사망 시 발생합니다.</summary>
        public event Action OnDeath;

        // ── Unity 생명주기 ────────────────────────────────
        private void Awake()
        {
            _currentHealth = _maxHealth;
        }

        private void OnEnable()
        {
            if (_autoRegen)
            {
                _regenCoroutine = StartCoroutine(RegenRoutine());
            }
        }

        private void OnDisable()
        {
            if (_regenCoroutine != null)
            {
                StopCoroutine(_regenCoroutine);
                _regenCoroutine = null;
            }
        }

        // ── public 메서드 ─────────────────────────────────
        /// <inheritdoc/>
        public void TakeDamage(float amount)
        {
            if (!IsAlive) { return; }

            _currentHealth = Mathf.Max(0, _currentHealth - (int)amount);
            OnDamageTaken?.Invoke(amount);

            if (!IsAlive)
            {
                HandleDeath();
            }
        }

        /// <summary>체력을 지정한 양만큼 회복합니다.</summary>
        public void Heal(float amount)
        {
            if (!IsAlive) { return; }

            _currentHealth = Mathf.Min(_maxHealth, _currentHealth + (int)amount);
        }

        // ── private 메서드 ────────────────────────────────
        private void HandleDeath()
        {
            OnDeath?.Invoke();
            enabled = false;
        }

        private System.Collections.IEnumerator RegenRoutine()
        {
            while (IsAlive)
            {
                yield return s_regenWait;

                if (_currentHealth < _maxHealth)
                {
                    Heal(_regenRate);
                }
            }
        }

#if UNITY_EDITOR
        private void OnDrawGizmosSelected()
        {
            UnityEditor.Handles.Label(
                transform.position + Vector3.up * 2f,
                $"HP: {_currentHealth}/{_maxHealth}");
        }
#endif
    }
}
```