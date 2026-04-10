# Unity Test Framework 참조

Unity Test Framework(UTF)는 NUnit 3.5 기반의 Unity 공식 테스트 프레임워크다.
공식 문서: https://docs.unity3d.com/Packages/com.unity.test-framework@2.0/manual/index.html

---

## 테스트 파일 & 클래스 구성

### 규칙

- **파일** — 피처/도메인 단위로 하나
- **클래스** — 파일당 하나, `도메인명_테스트`
- **상태/조건** — 테스트 메서드명에 녹임
- **파일 분리** — 사용자가 요청할 때 나눔

### 폴더 구조 예시

```
Tests/
  EditMode/
    인벤토리_테스트.cs
    전투_테스트.cs
    상점_테스트.cs
  PlayMode/
    전투씬_테스트.cs
```

### 클래스 & 메서드 예시

```csharp
public class 인벤토리_테스트
{
    [Test] public void 빈_상태에서_아이템_추가하면_개수가_1이_된다() { }
    [Test] public void 빈_상태에서_아이템_제거하면_예외가_발생한다() { }
    [Test] public void 가득찬_상태에서_아이템_추가하면_실패한다() { }
}
```

Test Runner에서는 클래스 단위로 그룹화되어 표시된다:

```
✅ 인벤토리_테스트
   ✅ 빈_상태에서_아이템_추가하면_개수가_1이_된다
   ✅ 빈_상태에서_아이템_제거하면_예외가_발생한다
   ✅ 가득찬_상태에서_아이템_추가하면_실패한다
✅ 전투_테스트
   ✅ ...
```

> **중첩 클래스 주의**: Unity Test Framework는 중첩 클래스(`class WhenEmpty { }` 등)를 Test Runner UI에서 실행할 수 없다. 상태 구분은 메서드명으로 표현한다.

---

## 테스트 어셈블리 구성

### 어셈블리 생성

1. `Window > General > Test Runner` 열기
2. **EditMode** 탭 → `Create EditMode Test Assembly Folder`
3. **PlayMode** 탭 → `Create PlayMode Test Assembly Folder`

생성된 폴더 안에 `.asmdef` 파일이 자동으로 만들어진다.
테스트할 프로덕션 코드가 별도 어셈블리라면, 해당 `.asmdef`를 테스트 어셈블리의 **References**에 추가해야 한다.

### 권장 폴더 구조

```
Assets/
  Scripts/
    MyGame.asmdef               ← 프로덕션 코드
  Tests/
    EditMode/
      MyGame.Tests.EditMode.asmdef
    PlayMode/
      MyGame.Tests.PlayMode.asmdef
```

### EditMode `.asmdef` 핵심 설정

```json
{
  "name": "MyGame.Tests.EditMode",
  "references": ["MyGame"],
  "includePlatforms": ["Editor"],
  "optionalUnityReferences": ["TestAssemblies"]
}
```

`"includePlatforms": ["Editor"]` — Editor만 타겟으로 지정하면 EditMode 테스트로 동작한다.

### PlayMode `.asmdef` 핵심 설정

```json
{
  "name": "MyGame.Tests.PlayMode",
  "references": ["MyGame"],
  "includePlatforms": [],
  "optionalUnityReferences": ["TestAssemblies"]
}
```

`"includePlatforms": []` (빈 배열, Any Platform) — PlayMode 테스트로 동작한다.

---

## EditMode vs PlayMode 판단 기준

공식 문서: https://docs.unity3d.com/Packages/com.unity.test-framework@2.0/manual/edit-mode-vs-play-mode-tests.html

### EditMode

- Unity Editor 안에서만 실행된다
- 씬을 실행하지 않으므로 빠르고 안정적이다
- `EditorApplication.update` 콜백 루프에서 실행된다
- Editor 코드와 게임 코드 모두 접근 가능하다

**EditMode가 적합한 대상:**

| 대상 | 이유 |
|---|---|
| 순수 C# 로직 | Unity 런타임 불필요 |
| 도메인 규칙, 계산 로직 | Unity 런타임 불필요 |
| Fake/Mock으로 의존성 대체 가능한 코드 | Unity 런타임 불필요 |
| Editor 확장 (`CustomEditor`, `EditorWindow`) | Editor 코드 접근 가능 |
| ScriptableObject 데이터 로직 | 씬 실행 불필요 |

### PlayMode

- 씬을 실제로 실행하며 Unity 런타임에 의존한다
- `[UnityTest]`로 코루틴으로 실행된다
- EditMode보다 느리고 설정이 복잡하다

**PlayMode가 필요한 대상:**

| 대상 | 이유 |
|---|---|
| MonoBehaviour 생명주기 (`Awake`, `Start`, `Update`, `OnDestroy`) | 런타임 없이 호출 안 됨 |
| Physics (`Rigidbody`, `Collider`, 충돌 이벤트) | Physics 루프 필요 |
| Coroutine (`StartCoroutine`, `yield return`) | MonoBehaviour 필요 |
| Scene 로드 / 전환 (`SceneManager`) | 런타임 필요 |
| 실제 프레임 경과 (`Time.deltaTime`) | 프레임 루프 필요 |

> MonoBehaviour에서 로직을 순수 C# 클래스로 분리할 수 있다면, 분리 후 EditMode로 테스트하는 것이 우선이다.

### RequiresPlayMode 어트리뷰트

기본 동작을 재정의할 수 있다:

```csharp
// Editor 어셈블리지만 PlayMode에서 실행
[RequiresPlayMode]
public class MyPlayModeTest { }

// 플랫폼 어셈블리지만 EditMode에서 실행
[RequiresPlayMode(false)]
public class MyEditModeTest { }
```

---

## 테스트 어트리뷰트

### `[Test]` vs `[UnityTest]`

| 어트리뷰트 | 반환 타입 | 사용 시점 |
|---|---|---|
| `[Test]` | `void` | 기본. 프레임을 건너뛸 필요 없을 때 |
| `[UnityTest]` | `IEnumerator` | 프레임 건너뜀, 시간 대기, 코루틴이 필요할 때 |

공식 권장: **`[UnityTest]`가 필요하지 않으면 `[Test]`를 사용한다.**

```csharp
// 기본 동기 테스트
[Test]
public void 항목을_추가하면_개수가_증가한다()
{
    var collection = new ItemCollection();
    collection.Add(new Item());
    Assert.AreEqual(1, collection.Count);
}

// 프레임 대기가 필요한 테스트 (EditMode에서 프레임 건너뜀)
[UnityTest]
public IEnumerator 다음_프레임에_상태가_갱신된다()
{
    var sut = new GameObject().AddComponent<MyComponent>();
    sut.Trigger();

    yield return null; // 한 프레임 건너뜀

    Assert.IsTrue(sut.IsReady);
}
```

### SetUp / TearDown

```csharp
public class MyTests
{
    private ItemCollection _collection;

    [SetUp]
    public void SetUp()
    {
        _collection = new ItemCollection();
    }

    [TearDown]
    public void TearDown()
    {
        // PlayMode에서 생성한 GameObject는 여기서 정리
        // Object.Destroy(_gameObject);
    }

    [Test]
    public void 초기_상태는_비어있다()
    {
        Assert.IsTrue(_collection.IsEmpty());
    }
}
```

### UnitySetUp / UnityTearDown

SetUp/TearDown에서 `yield`가 필요한 경우 사용한다.

```csharp
[UnitySetUp]
public IEnumerator SetUp()
{
    yield return new EnterPlayMode();
}

[UnityTearDown]
public IEnumerator TearDown()
{
    yield return new ExitPlayMode();
}
```

### TestCase — 매개변수화 테스트

```csharp
[TestCase(0,   false)]
[TestCase(1,   true)]
[TestCase(100, true)]
public void 값에_따라_유효성이_달라진다(int value, bool expectedValid)
{
    var result = Validator.IsValid(value);
    Assert.AreEqual(expectedValid, result);
}
```

> **주의**: `[UnityTest]`는 `[TestCase]`를 지원하지 않는다(`[ValueSource]`만 지원). 매개변수화 테스트는 `[Test]`와 함께 사용한다.

### OneTimeSetUp / OneTimeTearDown

테스트 클래스 전체에서 한 번만 실행된다.

```csharp
[OneTimeSetUp]
public void OneTimeSetUp() { /* 비용이 큰 초기화 */ }

[OneTimeTearDown]
public void OneTimeTearDown() { /* 전체 정리 */ }
```

---

## 유니티 전용 Assert

`UnityEngine.TestTools.Utils` 네임스페이스에서 제공한다.

```csharp
// 부동소수점 비교 (기본 허용 오차 0.00001f)
Assert.AreApproximatelyEqual(0.1f, result);
Assert.AreApproximatelyEqual(expected, actual, tolerance: 0.001f);
Assert.AreNotApproximatelyEqual(0f, result);

// Vector, Color, Quaternion 비교
using UnityEngine.TestTools.Utils;

Assert.That(actual, Is.EqualTo(expected).Using(Vector3EqualityComparer.Instance));
Assert.That(actual, Is.EqualTo(expected).Using(new Vector3EqualityComparer(0.01f)));
Assert.That(color, Is.EqualTo(expected).Using(ColorEqualityComparer.Instance));
Assert.That(rot, Is.EqualTo(expected).Using(QuaternionEqualityComparer.Instance));
```

---

## PlayMode 코루틴 테스트

```csharp
// 시간 대기
[UnityTest]
public IEnumerator 일정_시간_후_완료된다()
{
    var sut = new GameObject().AddComponent<MyComponent>();
    sut.StartProcess();

    yield return new WaitForSeconds(1f);

    Assert.IsTrue(sut.IsComplete);
}

// WaitForFixedUpdate — Physics 결과 확인 시
[UnityTest]
public IEnumerator 물리_시뮬레이션_후_위치가_변경된다()
{
    var go = new GameObject();
    go.AddComponent<Rigidbody>();
    var originalY = go.transform.position.y;

    yield return new WaitForFixedUpdate();

    Assert.AreNotEqual(originalY, go.transform.position.y);
}
```

> PlayMode에서 `new GameObject()`로 생성한 오브젝트는 `[TearDown]`에서 `Object.Destroy()`로 반드시 정리한다.

---

## MonoBehaviourTest

MonoBehaviour를 직접 테스트해야 할 때 사용하는 UTF 헬퍼다.

```csharp
[UnityTest]
public IEnumerator MonoBehaviour_테스트_예시()
{
    yield return new MonoBehaviourTest<MyBehaviourTest>();
}

public class MyBehaviourTest : MonoBehaviour, IMonoBehaviourTest
{
    private int _frameCount;

    public bool IsTestFinished => _frameCount > 10;

    void Update() => _frameCount++;
}
```

---

## 테스트 실행

### Test Runner UI

`Window > General > Test Runner`

- **EditMode 탭** — EditMode 테스트만 표시
- **PlayMode 탭** — PlayMode 테스트만 표시
- `Run All` — 전체 실행
- `Run Selected` — 선택한 테스트만 실행
- 검색창으로 테스트 필터링 가능

### 커맨드라인 실행 (결과 XML 저장)

```bash
# EditMode 테스트 실행 후 결과를 XML로 저장
Unity.exe -batchmode -nographics \
  -runTests \
  -projectPath "PATH_TO_PROJECT" \
  -testPlatform EditMode \
  -testResults "PATH_TO_PROJECT/TestResults.xml"

# PlayMode
Unity.exe -batchmode \
  -runTests \
  -projectPath "PATH_TO_PROJECT" \
  -testPlatform PlayMode \
  -testResults "PATH_TO_PROJECT/TestResults.xml"
```

> `-quit` 플래그를 함께 사용하지 않는다. NUnit이 테스트 완료 후 자동으로 종료한다.

### 결과 XML 구조 (NUnit 포맷)

```xml
<test-suite result="Failed">
  <test-case name="항목을_추가하면_개수가_증가한다"
             result="Failed"
             fullname="MyGame.Tests.CollectionTests.항목을_추가하면_개수가_증가한다">
    <failure>
      <message>Expected: 1, But was: 0</message>
      <stack-trace>at CollectionTests... line 42</stack-trace>
    </failure>
  </test-case>
</test-suite>
```

AI가 XML 파일을 읽어 `result` 어트리뷰트와 `<failure>` 블록을 분석해 Red/Green 상태를 판단한다.

---

## MCP for Unity (선택 설치)

MCP for Unity를 설치하면 AI 클라이언트에서 테스트 실행과 결과 수신을 자동화할 수 있다.

**권장 패키지**: https://github.com/CoderGamester/mcp-unity

**설치:**
```
Unity Package Manager → + → Add package from git URL
https://github.com/CoderGamester/mcp-unity.git
```

**AI 클라이언트 설정** (`claude_desktop_config.json` 또는 `mcp.json`):
```json
{
  "mcpServers": {
    "mcp-unity": {
      "command": "node",
      "args": ["ABSOLUTE/PATH/TO/mcp-unity/Server~/build/index.js"]
    }
  }
}
```

설치 후 Unity에서 `Tools > MCP Unity > Server Window`를 열어 서버를 시작한다.

**연결 시 사용 가능한 도구:**
- 테스트 목록 조회 — `unity://tests/editmode`, `unity://tests/playmode`
- 테스트 실행 및 결과 수신 — `run_tests`
- Unity Console 로그 조회 — `get_console_logs`

---

## 모킹 라이브러리 (선택 설치)

인터페이스 Mock이 필요한 경우 **NSubstitute** 사용을 권장한다.

**NSubstitute** `Packages/manifest.json`에 추가:
```json
{
  "dependencies": {
    "com.nsubstitute.nsubstitute": "https://github.com/nsubstitute/NSubstitute.git#4.4.0"
  }
}
```

또는 NuGet에서 DLL을 받아 `Assets/Plugins/`에 배치한다.

테스트 어셈블리 `.asmdef`의 References에 `NSubstitute`를 추가한다.
