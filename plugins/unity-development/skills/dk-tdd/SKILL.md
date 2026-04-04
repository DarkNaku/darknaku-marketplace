---
name: dk-tdd
description: TDD 워크플로(Red-Green-Refactor)와 Tidy First 원칙에 따라 기능을 구현하는 스킬. 기능 개발 시 이 스킬을 사용하여 테스트 주도로 구현한다.
user-invocable: false
---

# Unity-TDD

켄트 벡의 TDD와 Tidy First 원칙을 따라 기능을 구현하는 스킬.
빠른 구현보다 **깔끔하고 테스트된 코드**를 우선한다.

---

## 전제 조건

- **인풋**: 구현할 기능 명세 (UseCase, Domain 로직 등)
- **아웃풋**: TDD 사이클을 거쳐 테스트와 함께 구현된 코드

---

## 핵심 원칙

- 항상 TDD 사이클을 따른다: **Red → Green → Refactor**
- 가장 간단하게 실패하는 테스트부터 작성한다
- 테스트를 통과하는 데 필요한 **최소한의 코드**만 구현한다
- 테스트가 통과된 후에만 리팩토링한다
- **구조적 변경과 동작적 변경을 절대 같은 커밋에 혼합하지 않는다**

---

## 워크플로우

### Step 1: 기능 분해

구현할 기능을 가장 작은 증분 단위로 분해한다.

- 각 증분은 하나의 테스트로 검증 가능해야 한다
- 증분 목록을 정리하고 구현 순서를 결정한다

---

### Step 2: Red — 실패하는 테스트 작성

- 현재 증분을 정의하는 실패 테스트를 작성한다
- 한 번에 테스트 하나만 작성한다
- 테스트 실패 이유가 명확하고 의미 있어야 한다
- 결함 수정 시: API 수준 실패 테스트 → 최소 재현 테스트 순으로 작성 후 둘 다 통과시킨다

**테스트 작성 규칙**:

#### 네이밍

테스트 메서드명은 **한국어 + 언더스코어**로 행동을 서술한다.

```csharp
// ❌ Java 스타일
[Test] public void shouldSumTwoPositiveNumbers() { }

// ✅ C# / Unity 관례
[Test] public void 양수_두개를_더하면_올바른_합을_반환한다() { }
[Test] public void 체력이_0이하면_사망_상태가_된다() { }
[Test] public void 공격력이_없으면_데미지를_주지_않는다() { }
```

#### EditMode vs PlayMode

| 테스트 대상 | 모드 | 이유 |
|---|---|---|
| Core — Domain, Application (UseCase) | **EditMode 우선** | Unity 런타임 미의존, 빠름 |
| Presentation — 순수 C# Presenter | **EditMode** | View 인터페이스 Fake 주입으로 가능 |
| Presentation — View, Infrastructure | PlayMode | Unity 런타임 필요 |

- **PlayMode 테스트는 꼭 필요한 경우에만 작성한다.** 느리기 때문에 EditMode로 커버 가능하면 EditMode를 선택한다.
- 테스트 실행 시 **PlayMode 테스트를 제외한 모든 테스트**를 매번 실행한다.

#### Fake / Mock 사용

- Repository 등 외부 의존은 **In-memory Fake 구현체**를 사용한다
- 단순 동작 검증이 필요하면 **NSubstitute**로 Mock을 생성한다
- Presenter 테스트 시 View 인터페이스의 Fake를 주입해 MonoBehaviour 없이 테스트한다

```csharp
// Fake Repository
public class FakePlayerRepository : IPlayerRepository
{
    private readonly Dictionary<int, Player> _store = new();
    public Player Find(int id) => _store.GetValueOrDefault(id);
    public void Save(Player player) => _store[player.Id] = player;
}

// NSubstitute Mock
var repo = Substitute.For<IPlayerRepository>();
repo.Find(1).Returns(new Player(1, 100));
```

---

### Step 3: Green — 최소한의 구현

- 테스트를 통과할 만큼만 코드를 작성한다. 그 이상은 작성하지 않는다
- 구현이 지저분해도 괜찮다. 정리는 Refactor 단계에서 한다
- 테스트를 실행해 Green을 확인한다

---

### Step 4: Refactor — 정리

- 테스트가 통과(Green)된 상태에서만 리팩토링한다
- 한 번에 하나의 리팩토링 변경만 수행한다
- 각 리팩토링 단계 후 테스트를 실행해 Green 상태를 유지한다
- 중복 제거와 명확성 개선을 우선한다

---

### Step 5: Tidy First — 구조적 변경

모든 변경을 두 가지로 구분한다:

| 유형 | 정의 | 예시 |
|---|---|---|
| **구조적 변경** | 동작은 그대로, 코드 구조만 재배치 | 이름 변경, 메서드 추출, 파일 이동 |
| **동작적 변경** | 실제 기능 추가 또는 수정 | 새 UseCase 구현, 버그 수정 |

- 구조적 변경과 동작적 변경이 모두 필요하면 **구조적 변경을 먼저** 진행한다
- 구조적 변경 전후에 테스트를 실행해 동작이 바뀌지 않았음을 확인한다

---

### Step 6: 커밋

다음 조건을 모두 만족할 때만 커밋한다:

1. 모든 테스트가 통과되었다
2. 모든 컴파일러 경고 및 Unity Console 오류가 해결되었다
3. 해당 변경은 단일 논리적 작업 단위를 나타낸다
4. 커밋 메시지에 구조적 변경인지 동작적 변경인지 명시한다

```
// 커밋 메시지 예시
[구조] PlayerRepository 메서드명 정리
[동작] 플레이어 공격 UseCase 구현
[동작] 체력 0 이하 사망 처리 버그 수정
```

크고 드문 커밋보다 **작고 잦은 커밋**을 선호한다.

---

### Step 7: 다음 증분 반복

- 다음 증분에 대해 Step 2부터 반복한다
- 기능이 완성될 때까지 반복한다

---

## 코드 품질 기준

- 중복을 제거한다
- 이름과 구조로 의도를 표현한다 — 주석 없이도 읽히도록
- 의존성을 명시적으로 드러낸다 (생성자 주입)
- 메서드는 하나의 책임에 집중하고 간결하게 유지한다
- 상태와 부작용을 최소화한다
- 가장 단순한 해결책을 선택한다

---

## 품질 체크리스트

- [ ] 모든 기능 증분이 실패 테스트(Red)부터 시작되었는가
- [ ] 각 테스트 통과 후 최소한의 코드만 구현되었는가
- [ ] 리팩토링은 Green 상태에서만 수행되었는가
- [ ] 구조적 변경과 동작적 변경이 분리되었는가
- [ ] 구조적 변경이 동작적 변경보다 먼저 수행되었는가
- [ ] 테스트명이 한국어로 조건·행위·기대값을 포함하는가
- [ ] EditMode 테스트를 우선 사용했는가
- [ ] 모든 커밋 시점에 전체 테스트가 통과하는가
