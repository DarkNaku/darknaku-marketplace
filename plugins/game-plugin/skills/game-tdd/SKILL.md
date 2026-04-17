---
name: game-tdd
description: TDD 워크플로(Red-Green-Refactor)와 Tidy First 원칙에 따라 기능을 구현하거나 버그를 수정하는 경우 사용합니다.
user-invocable: false
---

# TDD

켄트 벡의 TDD(Red-Green-Refactor)와 Tidy First 원칙을 따라 기능을 구현하는 스킬.
빠른 구현보다 **깔끔하고 테스트된 코드**를 우선한다.

---

## 핵심 철학

### 테스트는 동작(Behavior)을 검증한다, 구현(Implementation)이 아니라

좋은 테스트는 public interface를 통해 시스템이 **무엇을 하는지**를 검증한다.
내부 구조가 전혀 바뀌어도 테스트는 살아남아야 한다.

```csharp
// ❌ 구현 테스트 — 내부 필드를 직접 확인
Assert.AreEqual(0, _target._count);

// ✅ 동작 테스트 — public interface로 확인
Assert.IsTrue(_target.IsEmpty());
```

**경고 신호**: 내부 이름을 바꿨을 뿐인데 테스트가 깨진다면, 그 테스트는 구현을 테스트하고 있는 것이다.

### Vertical Slice — 한 번에 하나씩

> ❌ **Horizontal Slicing 안티패턴** — 절대 하지 말 것

```
# 잘못된 방식 (Horizontal)
RED:   test1, test2, test3, test4  ← 테스트 몰아쓰기
GREEN: impl1, impl2, impl3, impl4  ← 구현 몰아쓰기

# 올바른 방식 (Vertical)
RED→GREEN: test1 → impl1
RED→GREEN: test2 → impl2
RED→GREEN: test3 → impl3
```

테스트를 몰아서 쓰면 **실제 동작이 아닌 상상의 동작**을 테스트하게 된다.
각 사이클에서 배운 것이 다음 테스트의 방향을 결정한다.

---

## 전제 조건

- **인풋**: 구현할 기능 명세
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

### Step 1: 계획

코드를 한 줄도 작성하기 전에 확인한다:

- [ ] 어떤 동작(behavior)을 테스트할 것인가?
- [ ] 모든 엣지케이스가 아닌 **핵심 경로**부터 시작한다

---

### Step 2: Tracer Bullet — 첫 번째 총알

첫 테스트는 시스템의 핵심 경로가 end-to-end로 연결됨을 확인하는 **Tracer Bullet**이다.
완벽한 케이스가 아니어도 된다 — **경로가 뚫리는 것**이 목적이다.

```csharp
[Test]
public void 아이템을_추가하면_컬렉션에_포함된다()
{
    var sut = new ItemCollection();
    var item = new Item("sword");

    sut.Add(item);

    Assert.IsTrue(sut.Contains(item));
}
```

---

### Step 3: Red — 실패하는 테스트 작성

- 현재 증분 하나를 정의하는 실패 테스트를 작성한다
- **한 번에 테스트 하나만** 작성한다
- 테스트 실패 이유가 명확해야 한다
- 버그 수정 시: 버그를 재현하는 테스트를 먼저 작성하고, 수정 후 통과시킨다

#### 테스트 네이밍

테스트 메서드명은 동작을 명확하게 서술한다.

사용하는 프로그래밍 언어가 메서드명에 비ASCII 문자를 지원하는 경우, **사용자의 언어로 작성**해 가독성을 높인다:

```csharp
// C# — 사용자가 한국어를 사용하는 경우
[Test] public void 항목이_없을때_비어있음을_반환한다() { }
[Test] public void 항목을_추가하면_개수가_증가한다() { }
```

지원하지 않는 경우 영어로 동일한 의도를 표현한다:

```python
def test_returns_empty_when_no_items(): pass
def test_count_increases_when_item_added(): pass
```

#### Fake / Mock 사용 원칙

- 외부 의존은 **In-memory Fake 구현체**를 우선 사용한다
- **내부 구현체를 직접 Mock하지 않는다** — 인터페이스(public contract)만 대상으로 한다

```csharp
public class FakeRepository : IRepository
{
    private readonly List<Item> _store = new();
    public void Save(Item item) => _store.Add(item);
    public Item Find(int id) => _store.FirstOrDefault(i => i.Id == id);
}
```

---

### Step 4: Red 확인

Red 단계의 실패는 작성한 코드로 충분히 예측 가능하므로, 실제 실행은 생략하고 바로 Green 단계로 넘어간다.

---

### Step 5: Green — 최소한의 구현

- 테스트를 통과할 만큼만 코드를 작성한다. **그 이상은 작성하지 않는다**
- 미래 테스트를 예측해 미리 구현하지 않는다
- 구현이 지저분해도 괜찮다 — 정리는 Refactor 단계에서 한다
- 테스트를 실행해 Green을 확인한다

---

### Step 6: Refactor — 정리

- 테스트가 통과(Green)된 상태에서만 리팩토링한다
- 한 번에 하나의 리팩토링 변경만 수행한다
- 각 리팩토링 후 테스트를 실행해 Green 상태를 유지한다
- **Red 상태에서는 절대 리팩토링하지 않는다**
- 중복 제거와 의도 명확화를 우선한다

---

### Step 7: Tidy First — 구조적 변경

모든 변경을 두 가지로 구분한다:

| 유형 | 정의 | 예시 |
|---|---|---|
| **구조적 변경** | 동작은 그대로, 코드 구조만 재배치 | 이름 변경, 메서드 추출, 파일 이동 |
| **동작적 변경** | 실제 기능 추가 또는 수정 | 새 기능 구현, 버그 수정 |

- 둘 다 필요하면 **구조적 변경을 먼저** 진행한다
- 구조적 변경 전후에 테스트를 실행해 동작이 바뀌지 않았음을 확인한다

---

### Step 8: 커밋

다음 조건을 모두 만족할 때만 커밋한다:

1. 모든 테스트가 통과되었다
2. 모든 컴파일러 경고가 해결되었다
3. 단일 논리적 작업 단위다
4. 커밋 메시지에 변경 유형을 명시한다

```
[구조] 메서드명 정리
[동작] 아이템 추가 기능 구현
[동작] 빈 컬렉션 접근 시 예외 처리 버그 수정
```

크고 드문 커밋보다 **작고 잦은 커밋**을 선호한다.

---

### Step 9: 다음 증분 반복

Step 3부터 반복하며 기능이 완성될 때까지 이어간다.

---

## 코드 품질 기준

- 중복을 제거한다
- 이름과 구조로 의도를 표현한다 — 주석 없이도 읽히도록
- 의존성을 명시적으로 드러낸다 (생성자 주입)
- 메서드는 하나의 책임에 집중한다
- 상태와 부작용을 최소화한다
- 가장 단순한 해결책을 선택한다

---

## 사이클별 체크리스트

각 Red→Green 사이클마다 확인한다:

- [ ] 테스트가 동작(behavior)을 서술하는가, 구현(implementation)이 아니라
- [ ] 테스트가 public interface만 사용하는가
- [ ] 내부 리팩토링 후에도 이 테스트는 살아남는가
- [ ] 이 테스트를 통과하는 데 필요한 최소 코드만 작성했는가
- [ ] 미래를 예측한 투기성 구현이 없는가

## 전체 품질 체크리스트

- [ ] 모든 기능 증분이 실패 테스트(Red)부터 시작되었는가
- [ ] Horizontal Slicing(테스트 몰아쓰기)을 하지 않았는가
- [ ] 각 테스트 통과 후 최소한의 코드만 구현되었는가
- [ ] 리팩토링은 Green 상태에서만 수행되었는가
- [ ] 구조적 변경과 동작적 변경이 분리되었는가
- [ ] 구조적 변경이 동작적 변경보다 먼저 수행되었는가
- [ ] 테스트명이 한국어로 조건·행위·기대값을 포함하는가
- [ ] 모든 커밋 시점에 전체 테스트가 통과하는가
