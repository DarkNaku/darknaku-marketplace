# R3 오퍼레이터 레퍼런스

## 팩토리 메서드 (Observable 생성)

### 값 생성

| 메서드 | 설명 | 예시 |
|---|---|---|
| `Return(T)` | 단일 값 발행 후 완료 | `Observable.Return(42)` |
| `Empty<T>()` | 즉시 완료 | `Observable.Empty<int>()` |
| `Never<T>()` | 아무것도 발행하지 않음 | `Observable.Never<int>()` |
| `Throw<T>(Exception)` | 즉시 에러 | `Observable.Throw<int>(ex)` |
| `Range(start, count)` | 정수 시퀀스 | `Observable.Range(0, 10)` |
| `Repeat(T, count)` | 값 반복 | `Observable.Repeat("a", 3)` |
| `Defer(Func<Observable<T>>)` | 구독 시 생성 | `Observable.Defer(() => GetSource())` |

### 시간/프레임 기반

| 메서드 | 설명 |
|---|---|
| `Timer(TimeSpan)` | 지정 시간 후 단일 발행 |
| `Timer(TimeSpan, TimeSpan)` | 초기 지연 + 주기적 발행 |
| `TimerFrame(int)` | 지정 프레임 후 단일 발행 |
| `Interval(TimeSpan)` | 주기적 발행 |
| `IntervalFrame(int)` | 프레임 주기적 발행 |
| `EveryUpdate()` | 매 Update 프레임 |
| `EveryValueChanged(source, selector)` | 프로퍼티 변경 감지 (폴링) |

### 이벤트/비동기 변환

| 메서드 | 설명 |
|---|---|
| `FromEvent(add, remove)` | C# event → Observable |
| `FromEventHandler(add, remove)` | EventHandler → Observable |
| `FromAsync(Func<CancellationToken, ValueTask>)` | async → Observable |
| `Create(Func<Observer<T>, IDisposable>)` | 커스텀 Observable |
| `CreateFrom(Func<CancellationToken, IAsyncEnumerable<T>>)` | IAsyncEnumerable → Observable |

### 결합 팩토리

| 메서드 | 설명 |
|---|---|
| `Merge(params Observable<T>[])` | 여러 소스 병합 |
| `Concat(params Observable<T>[])` | 순차 연결 |
| `CombineLatest(s1, s2, selector)` | 각 소스의 최신 값 결합 |
| `Zip(s1, s2, selector)` | 대응하는 순서끼리 결합 |
| `Race(params Observable<T>[])` | 가장 먼저 발행하는 소스만 |

---

## 오퍼레이터 (체이닝)

### 필터링

```csharp
source.Where(x => x > 0)                    // 조건 필터
source.OfType<Derived>()                     // 타입 필터
source.Distinct()                            // 전체 중복 제거
source.DistinctUntilChanged()                // 연속 중복 제거
source.DistinctUntilChanged(x => x.Id)       // 키 기반 연속 중복 제거
source.Take(5)                               // 처음 N개
source.TakeLast(3)                           // 마지막 N개
source.TakeUntil(otherObservable)            // 다른 Observable 발행까지
source.TakeUntil(cancellationToken)          // 취소까지
source.Skip(2)                               // 처음 N개 건너뛰기
source.SkipUntil(otherObservable)            // 다른 Observable 발행 후부터
source.SkipLast(1)                           // 마지막 N개 제외
```

### 변환

```csharp
source.Select(x => x * 2)                   // 값 변환
source.SelectMany(x => GetItems(x))          // Observable 평탄화
source.SelectMany(x => FetchAsync(x))        // Task 평탄화
source.Cast<Derived>()                       // 타입 캐스팅
source.Index()                               // (index, value) 튜플
source.Flatten()                             // Observable<Observable<T>> 평탄화
```

### 비동기 변환

```csharp
// AwaitOperation: Sequential, Drop, Switch, Parallel
source.SelectAwait(async (x, ct) => await Fetch(x, ct), AwaitOperation.Switch)
source.WhereAwait(async (x, ct) => await Validate(x, ct), AwaitOperation.Sequential)
source.SubscribeAwait(async (x, ct) => await Process(x, ct), AwaitOperation.Drop)
```

### 타이밍

```csharp
source.Delay(TimeSpan.FromSeconds(1))        // 시간 지연
source.DelayFrame(5)                         // 프레임 지연
source.Debounce(TimeSpan.FromMilliseconds(300)) // 연속 입력 억제
source.DebounceFrame(10)                     // 프레임 기반 억제
source.ThrottleFirst(TimeSpan.FromSeconds(1)) // 첫 값만 허용
source.ThrottleFirstFrame(30)                // 프레임 기반 첫 값
source.ThrottleLast(TimeSpan.FromSeconds(1)) // 구간 내 마지막 값
source.ThrottleLastFrame(30)                 // 프레임 기반 마지막 값
source.Sample(TimeSpan.FromSeconds(1))       // 주기적 샘플링
source.SampleFrame(60)                       // 프레임 기반 샘플링
source.Timeout(TimeSpan.FromSeconds(5))      // 타임아웃
```

### 결합

```csharp
source.WithLatestFrom(other, (a, b) => (a, b))  // 최신 값과 결합
source.Zip(other, (a, b) => (a, b))              // 순서쌍 결합
source.CombineLatest(other, (a, b) => (a, b))    // 양쪽 최신 값
source.Merge(other)                               // 병합
source.Concat(other)                              // 순차 연결
source.StartWith(initialValue)                    // 초기값 선행
```

### 집계

```csharp
source.Count()                               // 개수 (완료 시 발행)
source.Sum()                                 // 합계
source.Average()                             // 평균
source.Min() / source.Max()                  // 최소/최대
source.Aggregate((acc, x) => acc + x)        // 리듀스
await source.FirstAsync()                    // 첫 값 (async)
await source.LastAsync()                     // 마지막 값 (async)
```

### 에러 처리

```csharp
source.Catch<T, TException>(ex => fallback)  // 에러 시 대체 Observable
source.Retry()                               // 에러 시 재구독
source.Retry(3)                              // 최대 N회 재구독
source.OnErrorResumeAsFailure()              // OnErrorResume → 실패 종료로 변환
```

### 유틸리티

```csharp
source.Do(x => Log(x))                      // 사이드 이펙트 (디버깅용)
source.DoOnCompleted(_ => Cleanup())         // 완료 시 사이드 이펙트
source.Chunk(5)                              // N개씩 묶기
source.Chunk(TimeSpan.FromSeconds(1))        // 시간 구간 묶기
source.Pairwise()                            // (이전값, 현재값) 쌍
source.Scan((acc, x) => acc + x)             // 누적 (매 값마다 발행)
source.AsSystemObservable()                  // IObservable<T>로 변환
```

---

## 오퍼레이터 선택 가이드

### "입력이 너무 빠르다" → 타이밍 오퍼레이터

| 상황 | 오퍼레이터 |
|---|---|
| 검색창 입력 (입력 멈추면 검색) | `Debounce` |
| 버튼 연타 방지 (첫 클릭만 허용) | `ThrottleFirst` |
| 스크롤 위치 업데이트 (마지막 위치만 필요) | `ThrottleLast` |
| 주기적 상태 확인 (1초마다 샘플링) | `Sample` |
| 일정 시간 응답 없으면 에러 | `Timeout` |

### "여러 소스를 하나로" → 결합 오퍼레이터

| 상황 | 오퍼레이터 |
|---|---|
| HP + MaxHP → 비율 계산 | `CombineLatest` |
| 클릭 시점의 현재 설정값 필요 | `WithLatestFrom` |
| 요청-응답 쌍 매칭 | `Zip` |
| 여러 이벤트를 하나로 수신 | `Merge` |
| 순서대로 실행 (A 완료 → B 시작) | `Concat` |

### "비동기 작업과 연동" → Await 오퍼레이터

| 상황 | AwaitOperation |
|---|---|
| API 호출 순서 보장 | `Sequential` |
| 버튼 클릭 → API 호출 (중복 방지) | `Drop` |
| 검색 자동완성 (이전 요청 취소) | `Switch` |
| 독립적인 병렬 처리 | `Parallel` |
