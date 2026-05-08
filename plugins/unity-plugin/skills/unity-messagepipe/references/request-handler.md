# Request/Response 패턴 상세

## 개요

MessagePipe의 Request/Response는 Mediator 패턴을 구현한다.
요청 메시지를 보내면 등록된 핸들러가 처리하고 응답을 반환한다.

---

## 동기 핸들러

### 단일 핸들러 (IRequestHandler)

하나의 요청에 하나의 핸들러가 응답한다.

```csharp
// 메시지 정의
public readonly struct CalculateDamageRequest
{
    public readonly int BaseDamage;
    public readonly float CritMultiplier;
    public readonly bool IsCrit;

    public CalculateDamageRequest(int baseDamage, float critMultiplier, bool isCrit)
    {
        BaseDamage = baseDamage;
        CritMultiplier = critMultiplier;
        IsCrit = isCrit;
    }
}

public readonly struct DamageResult
{
    public readonly int FinalDamage;
    public DamageResult(int finalDamage) => FinalDamage = finalDamage;
}

// 핸들러 구현
public class DamageCalculator : IRequestHandler<CalculateDamageRequest, DamageResult>
{
    public DamageResult Invoke(CalculateDamageRequest request)
    {
        var damage = request.IsCrit
            ? (int)(request.BaseDamage * request.CritMultiplier)
            : request.BaseDamage;
        return new DamageResult(damage);
    }
}

// 등록
builder.RegisterRequestHandler<CalculateDamageRequest, DamageResult, DamageCalculator>(options);

// 사용
public class CombatSystem
{
    private readonly IRequestHandler<CalculateDamageRequest, DamageResult> _damageCalc;

    public int CalculateDamage(int baseDamage, bool isCrit)
    {
        var result = _damageCalc.Invoke(
            new CalculateDamageRequest(baseDamage, 2.0f, isCrit));
        return result.FinalDamage;
    }
}
```

### 다중 핸들러 (IRequestAllHandler)

하나의 요청에 여러 핸들러가 응답한다. 유효성 검증이나 플러그인 시스템에 적합하다.

```csharp
// 여러 유효성 검사기
public class NameValidator : IRequestHandler<ValidatePlayerName, ValidationResult>
{
    public ValidationResult Invoke(ValidatePlayerName request)
    {
        if (string.IsNullOrEmpty(request.Name))
            return ValidationResult.Fail("이름이 비어있습니다");
        return ValidationResult.Ok();
    }
}

public class LengthValidator : IRequestHandler<ValidatePlayerName, ValidationResult>
{
    public ValidationResult Invoke(ValidatePlayerName request)
    {
        if (request.Name.Length < 2 || request.Name.Length > 20)
            return ValidationResult.Fail("이름은 2~20자여야 합니다");
        return ValidationResult.Ok();
    }
}

public class ProfanityValidator : IRequestHandler<ValidatePlayerName, ValidationResult>
{
    public ValidationResult Invoke(ValidatePlayerName request)
    {
        if (ContainsProfanity(request.Name))
            return ValidationResult.Fail("부적절한 단어가 포함되어 있습니다");
        return ValidationResult.Ok();
    }
}

// 등록 (각각 등록)
builder.RegisterRequestHandler<ValidatePlayerName, ValidationResult, NameValidator>(options);
builder.RegisterRequestHandler<ValidatePlayerName, ValidationResult, LengthValidator>(options);
builder.RegisterRequestHandler<ValidatePlayerName, ValidationResult, ProfanityValidator>(options);

// 사용 (IRequestAllHandler로 주입)
public class RegistrationPresenter
{
    private readonly IRequestAllHandler<ValidatePlayerName, ValidationResult> _validators;

    public bool ValidateName(string name)
    {
        var results = _validators.InvokeAll(new ValidatePlayerName(name));
        return results.All(r => r.IsValid);
    }
}
```

---

## 비동기 핸들러

### IAsyncRequestHandler

```csharp
public class CloudSaveHandler : IAsyncRequestHandler<SaveRequest, SaveResult>
{
    private readonly ICloudService _cloud;

    public CloudSaveHandler(ICloudService cloud) => _cloud = cloud;

    public async ValueTask<SaveResult> InvokeAsync(
        SaveRequest request, CancellationToken ct = default)
    {
        try
        {
            await _cloud.UploadAsync(request.Data, ct);
            return new SaveResult(true, "클라우드 저장 완료");
        }
        catch (Exception ex)
        {
            return new SaveResult(false, ex.Message);
        }
    }
}

// 등록
builder.RegisterAsyncRequestHandler<SaveRequest, SaveResult, CloudSaveHandler>(options);

// 사용
public class SavePresenter
{
    private readonly IAsyncRequestHandler<SaveRequest, SaveResult> _saveHandler;

    public async UniTask Save()
    {
        var result = await _saveHandler.InvokeAsync(new SaveRequest("data"));
        Debug.Log(result.Message);
    }
}
```

---

## RequestHandler 필터

### RequestHandlerFilter (동기)

요청 처리 전후에 공통 로직을 삽입한다.

```csharp
public class LoggingRequestFilter<TReq, TRes> : RequestHandlerFilter<TReq, TRes>
{
    public override TRes Invoke(TReq request, Func<TReq, TRes> next)
    {
        Debug.Log($"Request: {typeof(TReq).Name}");
        var response = next(request);
        Debug.Log($"Response: {typeof(TRes).Name}");
        return response;
    }
}

// 성능 측정 필터
public class PerformanceRequestFilter<TReq, TRes> : RequestHandlerFilter<TReq, TRes>
{
    public override TRes Invoke(TReq request, Func<TReq, TRes> next)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        var response = next(request);
        sw.Stop();

        if (sw.ElapsedMilliseconds > 16)
            Debug.LogWarning($"{typeof(TReq).Name} 처리 시간: {sw.ElapsedMilliseconds}ms");

        return response;
    }
}
```

### AsyncRequestHandlerFilter

```csharp
public class RetryFilter<TReq, TRes> : AsyncRequestHandlerFilter<TReq, TRes>
{
    public override async ValueTask<TRes> InvokeAsync(
        TReq request,
        CancellationToken ct,
        Func<TReq, CancellationToken, ValueTask<TRes>> next)
    {
        for (int i = 0; i < 3; i++)
        {
            try
            {
                return await next(request, ct);
            }
            catch when (i < 2)
            {
                await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: ct);
            }
        }

        return await next(request, ct);  // 마지막 시도는 예외 전파
    }
}
```

### 필터 적용 방법

```csharp
// 1. VContainer에 등록
builder.RegisterRequestHandlerFilter<LoggingRequestFilter<SaveRequest, SaveResult>>();

// 2. 핸들러 클래스에 어트리뷰트
[RequestHandlerFilter(typeof(LoggingRequestFilter<,>), Order = -100)]
public class SaveHandler : IRequestHandler<SaveRequest, SaveResult>
{
    // ...
}
```

> Unity(IL2CPP)에서는 오픈 제네릭 어트리뷰트를 사용할 수 없다.
> 구체 타입으로 `[RequestHandlerFilter(typeof(LoggingRequestFilter<SaveRequest, SaveResult>))]`처럼 지정한다.
