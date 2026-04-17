---
name: unity-test
description: Unity 프로젝트에서 테스트를 작성하거나 실행해야 할 때 반드시 사용합니다. TDD 사이클의 테스트 작성 및 실행 단계에서 Unity Test Framework(EditMode/PlayMode) 사용법, MCP 연동, 테스트 파일 구성 등 Unity 환경에 맞는 방식을 제공합니다. Unity에서 테스트와 관련된 작업이라면 이 스킬을 참조합니다.
user-invocable: false
---

# Unity Test

tdd 스킬의 절차를 따른다.
단, 테스트 작성과 실행은 이 스킬에서 정의한 Unity 방식을 따른다.

테스트 작성 상세 내용은 → [unity-test-reference.md](unity-test-reference.md) 참조

---

## 테스트 작성 방식

### EditMode vs PlayMode

**기본은 EditMode다.** PlayMode는 Unity 런타임이 반드시 필요한 경우에만 사용한다.

| EditMode로 테스트 | PlayMode로 테스트 |
|---|---|
| 순수 C# 로직 | MonoBehaviour 생명주기 (`Awake`, `Start`, `Update`) |
| 외부 의존이 없는 도메인 규칙 | Physics 시뮬레이션 |
| Fake/Mock으로 대체 가능한 의존성 | Coroutine 실행 |
| Editor 확장 | Scene 로드 / 전환 |

MonoBehaviour에서 로직을 순수 C# 클래스로 분리할 수 있다면, 분리 후 EditMode로 테스트하는 것이 우선이다.

### 모킹

외부 의존성이 있을 때는 우선 **In-memory Fake 구현체**로 직접 작성한다.
Mock 라이브러리가 필요한 상황이 생기면 사용자에게 알리고 필요 여부를 확인한다.

> Mock 라이브러리(예: NSubstitute)를 사용하면 인터페이스 의존성을 더 간결하게 다룰 수 있습니다. 설치가 필요하시면 안내해 드릴게요.

---

## 테스트 실행 방식

### Red 확인 (tdd Step 4 적용)

Red 단계의 실패는 작성한 코드로 충분히 예측 가능하므로 실행을 생략한다.

**MCP for Unity가 연결된 경우** — 테스트를 직접 실행해 Red를 확인한 후 진행한다.

> MCP for Unity를 사용하면 테스트 실행과 결과 수신을 자동화할 수 있어 개발 루프가 빨라집니다. 설치가 필요하시면 안내해 드릴게요.

### Green 확인 (tdd Step 5 적용)

**MCP for Unity가 연결된 경우** — 테스트를 직접 실행해 Green을 확인한다.

**MCP가 없는 경우** — 사용자에게 실행을 요청한다:

```
Test Runner(Window > General > Test Runner)에서 [테스트명]을 실행해 주세요.
완료되면 결과를 알려주시거나, -testResults 옵션으로 저장된 XML 파일 경로를 알려주세요.
```

사용자가 완료를 통지하면 XML 결과 파일을 읽어 Pass/Fail을 확인한다.
XML 구조 및 분석 방법 → [unity-test-reference.md](unity-test-reference.md) 참조

---

## 체크리스트 (Unity 추가 항목)

tdd 스킬의 체크리스트에 아래 항목을 추가로 확인한다:

- [ ] EditMode로 먼저 테스트하려 시도했는가
- [ ] PlayMode를 쓴 경우 Unity 런타임이 반드시 필요한 이유가 있는가
- [ ] Unity Console 오류가 없는가
