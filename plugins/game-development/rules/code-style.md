Coding Style Guide

Claude Code가 코드 생성 시 반드시 준수해야 할 C# / Unity 코딩 스타일 규칙입니다.


1. 언어 및 표기

주석, TODO/FIXME 태그, 문서는 한국어로 작성한다.
클래스명, 메서드명, 변수명 등 식별자는 영어로 작성한다.


2. 네이밍 컨벤션
대상규칙예시클래스 / 인터페이스 / 메서드PascalCasePlayerController, IRepository, GetScore()지역 변수 / 매개변수camelCaseplayerScore, deltaTimeprivate 필드_ 프리픽스 + camelCase_health, _speed상수 / enum 값모두 대문자 + _ 구분MAX_HEALTH, GAME_OVER프로퍼티PascalCaseIsAlive, CurrentScore

3. 코드 길이 및 복잡도

간결함을 우선하되, 가독성을 희생하지 않는다.
조건이 중첩되면 Early return으로 depth를 줄인다.
메서드는 한 가지 역할만 수행한다. 길어지면 분리한다.

csharp// ❌ 중첩 depth
public void Execute(Player player)
{
    if (player != null)
    {
        if (player.IsAlive)
        {
            player.TakeDamage(10);
        }
    }
}

// ✅ Early return
public void Execute(Player player)
{
    if (player == null || !player.IsAlive) return;
    player.TakeDamage(10);
}

4. 주석

주석은 최소화한다. 코드 자체가 의도를 드러내도록 네이밍에 투자한다.
인라인 주석(//)은 "왜(Why)" 를 설명할 때만 사용한다. "무엇(What)"은 코드로 표현한다.
/// XML 문서 주석은 사용하지 않는다.
TODO / FIXME 태그는 적극 활용한다.

csharp// ❌ 코드를 반복하는 주석
// 플레이어 체력을 10 감소
player.TakeDamage(10);

// ✅ 이유를 설명하는 주석
// 첫 프레임은 델타타임이 비정상적으로 크므로 스킵
if (Time.frameCount <= 1) return;

// TODO: 난이도별 데미지 계수 적용 필요
// FIXME: 체력이 0 이하일 때 중복 호출 가능

5. var 키워드

타입이 우변에서 명확히 드러나면 var를 사용한다.
타입이 모호하거나 가독성이 떨어지면 명시적으로 선언한다.

csharp// ✅ 우변에서 타입이 명확
var player = new Player();
var scores = new List<int>();

// ❌ 우변만 보고 타입을 알 수 없음
var result = GetData();

6. C# 문법 스타일
가독성을 해치지 않는 선에서 아래 문법을 자유롭게 사용한다.
Expression-bodied 메서드
csharppublic bool IsAlive => _health > 0;
public int GetScore() => _score;
Null 조건 연산자
csharpvar name = player?.Name ?? "Unknown";
player?.TakeDamage(10);
패턴 매칭 / switch expression
csharpvar message = state switch
{
    GameState.PLAYING => "게임 중",
    GameState.PAUSED  => "일시정지",
    _                 => "알 수 없음"
};
record / init 프로퍼티 (불변 데이터 모델에 사용)
csharppublic record PlayerData(string Name, int Score);

public class Config
{
    public int MaxHealth { get; init; }
}

특정 문법 사용이 목적이 아니라, 가독성 좋고 짧은 코드가 목적임을 명심한다.


7. 클래스 멤버 선언 순서
접근제한자 기준으로 정렬한다.
public    → 상수, 이벤트, 프로퍼티, 메서드
protected → 필드, 메서드
private   → 필드, 메서드
csharppublic class Enemy
{
    // public
    public event Action OnDied;
    public int Score { get; private set; }

    // protected
    protected float _moveSpeed;

    // private
    private int _health;
    private Rigidbody _rigidbody;

    // Unity 생명주기 메서드는 private 메서드 그룹 최상단
    private void Awake() { ... }
    private void Update() { ... }

    public void TakeDamage(int amount) { ... }
    private void Die() { ... }
}

8. Unity 특화 규칙
SerializeField

인스펙터에서 사용자가 확인/변경해야 하는 필드는 [SerializeField] private으로 선언한다.
런타임에만 사용하는 내부 필드는 [SerializeField] 불필요.

csharp// ✅
[SerializeField] private float _moveSpeed = 5f;
[SerializeField] private AudioClip _jumpSound;

// ❌ public 필드로 노출하지 않는다
public float moveSpeed = 5f;
컴포넌트 캐싱

GetComponent는 반드시 Awake에서 캐싱한다. Update/로직 메서드에서 호출 금지.

csharpprivate Rigidbody _rigidbody;
private Animator _animator;

private void Awake()
{
    _rigidbody = GetComponent<Rigidbody>();
    _animator = GetComponent<Animator>();
}
비동기 처리

코루틴 대신 UniTask를 사용한다.

csharp// ❌ 코루틴
private IEnumerator DelayedAction()
{
    yield return new WaitForSeconds(1f);
    Execute();
}

// ✅ UniTask
private async UniTaskVoid DelayedAction()
{
    await UniTask.Delay(TimeSpan.FromSeconds(1f));
    Execute();
}