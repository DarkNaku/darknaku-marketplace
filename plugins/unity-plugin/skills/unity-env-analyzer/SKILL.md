---
name: unity-env-analyzer
description: Unity 프로젝트 전용 환경 분석 스킬. ProjectSettings/ProjectVersion.txt 또는 Packages/manifest.json이 존재하는 경우 이 스킬을 반드시 사용할 것 — 다른 스킬이 실행 중이더라도 이 마커 파일들이 감지되면 즉시 이 스킬을 이어서 실행한다. Unity 버전, 렌더 파이프라인, UPM 패키지, C# 사용 가능 기능을 분석하여 DEV_ENV.md를 생성한다.
---

# Unity 개발 환경 분석기

Unity 프로젝트를 분석하여 코드 작성 시 참고할 수 있는 상세한 개발 환경 문서를 생성한다.

> **출력 템플릿**: `templates/env-template.md` 참고  
> 문서 생성 전 반드시 해당 파일을 읽어 섹션 구조와 플레이스홀더를 확인할 것.

---

## 분석 순서

### 1단계: 프로젝트 루트 확인

사용자가 경로를 명시하지 않으면 업로드된 파일 위치(`/mnt/user-data/uploads/`) 또는 현재 디렉터리에서 탐색한다.

```bash
find . -name "ProjectVersion.txt" -path "*/ProjectSettings/*" 2>/dev/null | head -5
```

---

### 2단계: 핵심 파일 읽기

아래 파일을 순서대로 읽는다. 없으면 해당 항목 "확인 불가"로 표기.

| 파일 | 추출 정보 |
|------|-----------|
| `ProjectSettings/ProjectVersion.txt` | Unity 에디터 버전 |
| `Packages/manifest.json` | UPM 패키지 선언 목록 |
| `Packages/packages-lock.json` | 실제 설치된 버전 **(있으면 manifest보다 우선)** |
| `ProjectSettings/ProjectSettings.asset` | 스크립팅 백엔드, API 호환성, Input System, Define Symbols |
| `ProjectSettings/GraphicsSettings.asset` | 렌더 파이프라인 |
| `Assets/**/*.asmdef` | 어셈블리 정의 및 참조 구조 |

---

### 3단계: 각 파일에서 추출할 정보

#### ProjectVersion.txt
```
m_EditorVersion: 2021.3.15f1   ← 이 줄 추출
```

#### packages-lock.json (우선) / manifest.json (fallback)
```json
{
  "dependencies": {
    "com.unity.render-pipelines.universal": {
      "version": "12.1.7"   ← packages-lock.json의 실제 설치 버전
    }
  }
}
```

#### ProjectSettings.asset 추출 키
- `scriptingBackend` → 0=Mono, 1=IL2CPP
- `apiCompatibilityLevelPerPlatform` 또는 `apiCompatibilityLevel`
  - 2 = .NET Standard 2.0 / 3 = .NET Standard 2.1 / 6 = .NET 4.x
- **`activeInputHandler`** → **0=Legacy, 1=New Input System, 2=Both 동시 사용**
- `scriptingDefineSymbols` → 커스텀 Define 심볼
- `enabledVRDevices`, `xrSettings` → XR 사용 여부

#### GraphicsSettings.asset 추출 키
- `m_CustomRenderPipeline` 값 없음 → Built-in
- 경로에 "Universal" 포함 → URP
- 경로에 "HighDefinition" 포함 → HDRP

---

### 4단계: C# 버전 및 사용 가능/불가 기능 매핑

Unity 버전으로부터 C# 버전을 결정하고, **사용 가능/불가 기능을 모두 동적으로 생성**한다.

| Unity 버전 | C# 버전 |
|------------|---------|
| 2019.x | C# 7.3 |
| 2020.x | C# 8.0 |
| 2021.x ~ 6000.x | C# 9.0 |

**버전별 사용 가능 기능 목록:**

| 기능 | 최소 C# 버전 | 예시 |
|------|-------------|------|
| `ValueTuple`, 분해 | 7.0 | `var (x, y) = pos;` |
| Pattern matching 기초 (`is`, `switch`) | 7.0 | `if (obj is Enemy e)` |
| `ref returns`, `ref locals` | 7.0 | `ref int Get() => ref arr[0];` |
| `out` 변수 인라인 선언 | 7.0 | `TryGet(out var val)` |
| `throw` expressions | 7.0 | `val ?? throw new ...` |
| 로컬 함수 | 7.0 | `void Inner() { }` |
| `default` 리터럴 | 7.1 | `T val = default;` |
| `async Main` | 7.1 | - |
| 비null 참조 패턴 | 7.2 | `in` 매개변수 |
| `readonly struct` | 7.2 | `readonly struct Vector` |
| `private protected` | 7.2 | - |
| `stackalloc` 초기화 | 7.3 | `Span<int> s = stackalloc int[4];` |
| 특성의 `Enum`, `Delegate` 제약 | 7.3 | - |
| Nullable Reference Types | 8.0 | `string? name;` |
| Switch expressions | 8.0 | `x switch { 1 => "one", _ => "?" }` |
| `using var` | 8.0 | `using var conn = ...;` |
| Range / Index 연산자 | 8.0 | `arr[^1]`, `arr[1..3]` |
| `??=` null 병합 할당 | 8.0 | `list ??= new();` |
| `Span<T>` / `Memory<T>` | 8.0 + .NET Std 2.1 | 고성능 메모리 슬라이싱 |
| Records (`record class`) | 9.0 | `record class Point(int X, int Y);` |
| Target-typed `new()` | 9.0 | `List<int> list = new();` |
| Logical patterns (`and/or/not`) | 9.0 | `hp is > 0 and <= 30` |
| `init` only setters | 9.0 | `public int X { get; init; }` |
| Pattern matching 강화 | 9.0 | 타입 패턴, 관계형 패턴 |

**API 호환성 레벨에 따른 추가 제약:**
- `.NET Standard 2.0` → `Span<T>` 사용 불가, `HashSet` 일부 API 없음
- `.NET Standard 2.1` → `Span<T>`, `Memory<T>`, `IAsyncEnumerable` 사용 가능
- `.NET 4.x` → 가장 넓은 API 접근, Editor 전용 빌드에 권장

**사용 불가 기능은 감지된 C# 버전 기준으로 동적으로 결정한다:**
- C# 9.0이면 → C# 10+ 기능이 불가 목록
- C# 8.0이면 → C# 9.0+ 기능이 불가 목록 (records, target-typed new 등 포함)
- C# 7.3이면 → C# 8.0+ 기능이 불가 목록 (Nullable Reference Types, switch expressions 등 포함)

---

### 5단계: 패키지 분류 및 주요 API 정보 기재

`packages-lock.json` (없으면 `manifest.json`)의 패키지를 분류하고, 코드 작성에 필요한 네임스페이스와 주요 클래스를 함께 기재한다.

| 분류 | 패키지 ID | 주요 네임스페이스 / API |
|------|-----------|------------------------|
| 렌더 파이프라인 | `com.unity.render-pipelines.universal` | `UnityEngine.Rendering.Universal` |
| 렌더 파이프라인 | `com.unity.render-pipelines.high-definition` | `UnityEngine.Rendering.HighDefinition` |
| DOTS / ECS | `com.unity.entities` | `Unity.Entities` |
| DOTS / Burst | `com.unity.burst` | `Unity.Burst`, `[BurstCompile]` |
| DOTS / Collections | `com.unity.collections` | `Unity.Collections`, `NativeArray<T>` |
| DOTS / Jobs | `com.unity.jobs` | `Unity.Jobs`, `IJob`, `IJobParallelFor` |
| 수학 | `com.unity.mathematics` | `Unity.Mathematics`, `float3`, `math.*` |
| UI Toolkit | `com.unity.ui` | `UnityEngine.UIElements` |
| uGUI | `com.unity.ugui` | `UnityEngine.UI` |
| TextMeshPro | `com.unity.textmeshpro` | `TMPro`, `TextMeshProUGUI` |
| 입력 | `com.unity.inputsystem` | `UnityEngine.InputSystem` |
| Addressables | `com.unity.addressables` | `UnityEngine.AddressableAssets` |
| Cinemachine | `com.unity.cinemachine` | `Cinemachine` |
| 물리 | `com.unity.physics` | `Unity.Physics` |
| 네트워크 | `com.unity.netcode.gameobjects` | `Unity.Netcode` |
| 테스트 | `com.unity.test-framework` | `NUnit.Framework`, `UnityEngine.TestTools` |

분류되지 않은 패키지는 "기타 패키지" 섹션에 나열한다.  
`file:` 또는 `path:` 접두사 패키지는 "로컬 커스텀 패키지"로 별도 표기한다.

---

### 6단계: Asset Store 패키지 탐지

UPM 외 방식으로 설치된 패키지를 탐지한다.

```bash
# Assets/ 하위에서 package.json을 가진 폴더 탐색 (Asset Store 패키지 관행)
find ./Assets -maxdepth 3 -name "package.json" 2>/dev/null

# 주요 Asset Store 패키지 폴더명으로 식별
find ./Assets -maxdepth 2 -type d \( \
  -name "DOTween" -o -name "Demigiant" \
  -o -name "UniRx" \
  -o -name "Odin*" \
  -o -name "Sirenix" \
  -o -name "Photon*" \
  -o -name "Mirror" \
  -o -name "Easy*" \
  -o -name "Lean*" \
  -o -name "Feel" \
  -o -name "MoreMountains" \
\) 2>/dev/null
```

발견된 Asset Store 패키지는 버전 확인을 위해 해당 폴더의 `package.json` 또는 README를 읽는다.  
버전을 확인할 수 없으면 "버전 미확인"으로 표기한다.

**탐지된 주요 패키지별 네임스페이스:**

| 패키지 | 네임스페이스 |
|--------|-------------|
| DOTween | `DG.Tweening` |
| UniRx | `UniRx` |
| Photon PUN | `Photon.Pun`, `Photon.Realtime` |
| Mirror | `Mirror` |
| MoreMountains Feel | `MoreMountains.Feedbacks` |

---

### 7단계: NuGet 패키지 탐지

#### 방법 1: NuGetForUnity (`packages.config`)
```bash
find . -maxdepth 2 -name "packages.config" 2>/dev/null
```
발견 시 → 패키지명 + 버전 추출.

#### 방법 2: 수동 DLL (`Assets/Plugins/`)
```bash
find . -path "*/Assets/Plugins*" -name "*.dll" 2>/dev/null
```
- `packages.config`에 이미 등록된 DLL은 중복 표기 생략
- 버전 확인 불가 시 "버전 미확인"으로 표기

NuGet 패키지가 전혀 없으면 해당 섹션을 "해당 없음"으로 채운다.

---

### 8단계: Input System 정보 기재

`activeInputHandler` 값에 따라 아래 내용을 문서에 기재한다:

| 값 | 방식 | 코드 작성 가이드 |
|----|------|----------------|
| 0 | Legacy Input Manager | `Input.GetAxis()`, `Input.GetKey()`, `Input.GetMouseButton()` 사용 |
| 1 | New Input System | `using UnityEngine.InputSystem;` 필수. `InputAction`, `PlayerInput` 컴포넌트 활용 |
| 2 | Both | 두 방식 모두 사용 가능. 신규 코드는 New Input System 권장 |

---

### 9단계: 템플릿으로 문서 생성

`templates/env-template.md`를 읽고 아래 플레이스홀더를 실제 분석값으로 교체하여 최종 문서를 생성한다:

| 플레이스홀더 | 채울 내용 |
|-------------|-----------|
| `{{UNITY_VERSION}}` | Unity 에디터 버전 (예: `2021.3.15f1`) |
| `{{CSHARP_VERSION}}` | C# 버전 (예: `C# 9.0`) |
| `{{SCRIPTING_BACKEND}}` | Mono 또는 IL2CPP |
| `{{API_COMPATIBILITY}}` | .NET Standard 2.1 등 |
| `{{RENDER_PIPELINE}}` | URP / HDRP / Built-in + 버전 |
| `{{RENDER_PIPELINE_DETAIL}}` | 렌더 파이프라인 상세명 + 버전 |
| `{{RENDER_PIPELINE_NOTES}}` | 파이프라인별 코딩 주의사항 (아래 참고) |
| `{{CSHARP_AVAILABLE_FEATURES}}` | 4단계에서 결정된 사용 가능 기능 목록 (예시 코드 포함) |
| `{{CSHARP_UNAVAILABLE_FEATURES}}` | 4단계에서 결정된 사용 불가 기능 목록 |
| `{{PACKAGES_TABLE}}` | 5단계 분류된 UPM 패키지 테이블 (네임스페이스 포함) |
| `{{ASSET_STORE_TABLE}}` | 6단계 Asset Store 패키지 테이블 |
| `{{NUGET_SECTION}}` | 7단계 NuGet 패키지 테이블 또는 "해당 없음" |
| `{{INPUT_SYSTEM}}` | 8단계 Input System 방식 및 코드 가이드 |
| `{{ASMDEF_TABLE}}` | 어셈블리 구조 테이블 |
| `{{BUILD_TARGET}}` | 빌드 타겟 플랫폼 |
| `{{DEFINE_SYMBOLS}}` | 스크립팅 Define Symbols |
| `{{CODING_GUIDELINES}}` | 위 분석을 종합한 코딩 체크리스트 |

**`{{RENDER_PIPELINE_NOTES}}` 작성 기준:**

- **URP** → `ScriptableRendererFeature`로 Custom Pass 구현. `UniversalRenderPipeline.asset` 설정으로 품질 제어. Built-in의 `OnRenderImage` 사용 불가. Post Processing은 Volume 시스템 사용.
- **HDRP** → `CustomPass` 또는 `FullScreenCustomPass` 사용. `HDAdditionalCameraData`, `HDAdditionalLightData` 컴포넌트 필수. 볼류메트릭 등 고급 기능 사용 가능.
- **Built-in** → `OnRenderImage`, `Graphics.Blit` 사용 가능. `Camera.RenderToCubemap` 지원. Legacy Shaders 사용 가능.

저장 경로: `<프로젝트루트>/Docs/DEV_ENV.md` (`Docs` 폴더 없으면 자동 생성)  
Claude.ai에서 직접 쓰기 불가 시 → `/mnt/user-data/outputs/Docs/DEV_ENV.md`에 저장 후 경로 안내.

---

### 10단계: CLAUDE.md 업데이트

- `## 개발 환경` 섹션 없으면 → 파일 끝에 추가
- 섹션 이미 있으면 → 링크만 최신화 (중복 추가 금지)
- `CLAUDE.md` 없으면 → 새로 생성

추가할 내용:
```markdown
---

## 개발 환경

> Unity 프로젝트 개발 환경 정보는 아래 파일을 참고하세요.  
> 코드 작성 전 Unity 버전, C# 기능, 패키지 목록을 확인할 수 있습니다.

- [DEV_ENV.md](./Docs/DEV_ENV.md) — Unity 버전, 렌더 파이프라인, 패키지, C# 사용 가능 기능 정리
```

---

### 11단계: 완료 보고

- Unity 버전 및 C# 버전
- 렌더 파이프라인
- Input System 방식
- UPM 패키지 수, Asset Store 패키지 수, NuGet 패키지 수
- 주요 패키지 및 네임스페이스 목록
- 호환성 주의사항
- 저장된 파일 경로
