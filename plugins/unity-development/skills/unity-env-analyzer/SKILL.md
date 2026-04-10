---
name: unity-env-analyzer
description: Unity 프로젝트 폴더를 직접 분석하여 개발 환경 문서(UNITY_DEV_ENV.md)를 생성하는 스킬. 반드시 Unity 프로젝트(ProjectSettings/ProjectVersion.txt 또는 Packages/manifest.json이 존재하는 환경)에서만 동작하며, Unity 프로젝트가 아닌 경우 즉시 중단한다. 사용자가 "프로젝트 분석", "개발 환경 파악", "패키지 정리", "Unity 환경 문서화", "어떤 패키지 쓰고 있어?", "환경 파일 만들어줘" 등을 말할 때 반드시 이 스킬을 사용할 것. 업로드된 Unity 프로젝트 파일(manifest.json, ProjectSettings 등)이 있을 때도 즉시 사용할 것.
---

# Unity 개발 환경 분석기

Unity 프로젝트를 분석하여 코드 작성 시 참고할 수 있는 상세한 개발 환경 문서를 생성한다.

> **출력 템플릿**: `templates/env-template.md` 참고  
> 문서 생성 전 반드시 해당 파일을 읽어 섹션 구조와 플레이스홀더를 확인할 것.

---

## 분석 순서

### 0단계: Unity 프로젝트 여부 확인 (필수)

**이 스킬은 Unity 프로젝트에서만 동작한다.** 분석 시작 전 아래 파일 중 하나 이상이 존재하는지 확인한다:

```bash
find . -maxdepth 3 \( \
  -name "ProjectVersion.txt" -path "*/ProjectSettings/*" -o \
  -name "manifest.json" -path "*/Packages/*" \
\) 2>/dev/null | head -5
```

| 판별 기준 | 결과 |
|-----------|------|
| `ProjectSettings/ProjectVersion.txt` 존재 | Unity 프로젝트 확정 → 계속 |
| `Packages/manifest.json` 존재 | Unity 프로젝트 확정 → 계속 |
| 둘 다 없음 | **즉시 중단** |

Unity 프로젝트가 아닌 경우:
> "Unity 프로젝트를 찾을 수 없습니다. `ProjectSettings/ProjectVersion.txt` 또는 `Packages/manifest.json` 파일이 있는 Unity 프로젝트 폴더에서 실행해 주세요."

---

### 1단계: 프로젝트 루트 확인

사용자가 경로를 명시하지 않으면 업로드된 파일 위치(`/mnt/user-data/uploads/`) 또는 현재 디렉터리에서 탐색한다.

```bash
find . -name "ProjectVersion.txt" -path "*/ProjectSettings/*" 2>/dev/null | head -5
```

---

### 2단계: 핵심 파일 읽기

| 파일 | 추출 정보 |
|------|-----------|
| `ProjectSettings/ProjectVersion.txt` | Unity 에디터 버전 |
| `Packages/manifest.json` | UPM 패키지 전체 목록 + 버전 |
| `Packages/packages-lock.json` | 의존성 해결된 실제 설치 버전 (있으면 우선) |
| `ProjectSettings/ProjectSettings.asset` | 스크립팅 백엔드, API 호환성, Define Symbols |
| `ProjectSettings/GraphicsSettings.asset` | 렌더 파이프라인 Asset 참조 |
| `Assets/**/*.asmdef` | 어셈블리 정의 및 참조 구조 |

---

### 3단계: 각 파일에서 추출할 정보

#### ProjectVersion.txt
```
m_EditorVersion: 2021.3.15f1   ← 이 줄 추출
```

#### manifest.json
```json
{ "dependencies": { "com.unity.render-pipelines.universal": "12.1.7" } }
```
모든 dependencies 항목 추출. `"testables"` 섹션은 테스트 전용으로 별도 표기.

#### ProjectSettings.asset 추출 키
- `scriptingBackend` → 0=Mono, 1=IL2CPP
- `apiCompatibilityLevelPerPlatform` 또는 `apiCompatibilityLevel`
  - 2 = .NET Standard 2.0 / 3 = .NET Standard 2.1 / 6 = .NET 4.x
- `scriptingDefineSymbols` → 커스텀 Define 심볼
- `enabledVRDevices`, `xrSettings` → XR 사용 여부

#### GraphicsSettings.asset 추출 키
- `m_CustomRenderPipeline` 값 없음 → Built-in
- 경로에 "Universal" 포함 → URP
- 경로에 "HighDefinition" 포함 → HDRP

---

### 4단계: C# 버전 및 사용 가능 기능 매핑

| Unity 버전 | C# 버전 | 주요 사용 가능 기능 |
|------------|---------|-------------------|
| 2019.x | C# 7.3 | Pattern matching 기초, ref returns, ValueTuple |
| 2020.x | C# 8.0 | Nullable Reference Types, switch expressions, using declarations, ranges/indices |
| 2021.x | C# 9.0 | Records, target-typed new(), logical patterns (and/or/not), init-only setters |
| 6000.x (Unity 6) | C# 9.0 | 동일 (Unity는 아직 C# 10 미지원) |

**API 호환성 레벨에 따른 추가 제약:**
- `.NET Standard 2.0` → `Span<T>` 사용 제한적
- `.NET Standard 2.1` → `Span<T>`, `Memory<T>`, `IAsyncEnumerable` 사용 가능
- `.NET 4.x` → 가장 넓은 API 접근, Editor 전용 빌드에 권장

---

### 5단계: 패키지 분류

`manifest.json` 패키지를 아래 기준으로 분류하여 템플릿의 `{{PACKAGES_TABLE}}`을 채운다:

| 분류 | 패키지 ID |
|------|-----------|
| 렌더 파이프라인 | `com.unity.render-pipelines.universal`, `com.unity.render-pipelines.high-definition` |
| DOTS / ECS | `com.unity.entities`, `com.unity.burst`, `com.unity.collections`, `com.unity.jobs` |
| 물리 / 수학 | `com.unity.mathematics`, `com.unity.physics` |
| UI | `com.unity.ugui`, `com.unity.ui`, `com.unity.textmeshpro` |
| 입력 | `com.unity.inputsystem` |
| 애니메이션 | `com.unity.animation.rigging` |
| 테스트 | `com.unity.test-framework` |
| 기타 주요 | `com.unity.addressables`, `com.unity.cinemachine`, `com.unity.timeline`, `com.unity.localization`, `com.unity.netcode.gameobjects`, `com.unity.services.*` |

분류되지 않은 패키지는 "기타 패키지" 섹션에 나열한다.  
`file:` 또는 `path:` 접두사 패키지는 "로컬 커스텀 패키지"로 별도 표기한다.

---

### 6단계: NuGet 패키지 탐지

#### 방법 1: NuGetForUnity (`packages.config`)
```bash
find . -maxdepth 2 -name "packages.config" 2>/dev/null
```
발견 시 → 패키지명 + 버전 추출하여 `{{NUGET_SECTION}}` 채움.

#### 방법 2: 수동 DLL (`Assets/Plugins/`)
```bash
find . -path "*/Assets/Plugins*" -name "*.dll" 2>/dev/null
```
- `packages.config`에 이미 등록된 DLL은 중복 표기 생략
- 버전 확인 불가 시 "버전 미확인"으로 표기

NuGet 패키지가 전혀 없으면 `{{NUGET_SECTION}}`을 "해당 없음"으로 채운다.

---

### 7단계: 템플릿으로 문서 생성

`templates/env-template.md`를 읽고 아래 플레이스홀더를 실제 분석값으로 교체하여 최종 문서를 생성한다:

| 플레이스홀더 | 채울 내용 |
|-------------|-----------|
| `{{UNITY_VERSION}}` | Unity 에디터 버전 (예: `2021.3.15f1`) |
| `{{CSHARP_VERSION}}` | C# 버전 (예: `C# 9.0`) |
| `{{SCRIPTING_BACKEND}}` | Mono 또는 IL2CPP |
| `{{API_COMPATIBILITY}}` | .NET Standard 2.1 등 |
| `{{RENDER_PIPELINE}}` | URP / HDRP / Built-in + 버전 |
| `{{RENDER_PIPELINE_DETAIL}}` | 렌더 파이프라인 상세명 + 버전 |
| `{{RENDER_PIPELINE_NOTES}}` | 파이프라인별 코딩 주의사항 |
| `{{CSHARP_AVAILABLE_FEATURES}}` | 사용 가능한 C# 기능 목록 (예시 코드 포함) |
| `{{PACKAGES_TABLE}}` | 분류된 패키지 테이블 전체 |
| `{{NUGET_SECTION}}` | NuGet 패키지 테이블 또는 "해당 없음" |
| `{{ASMDEF_TABLE}}` | 어셈블리 구조 테이블 |
| `{{BUILD_TARGET}}` | 빌드 타겟 플랫폼 |
| `{{DEFINE_SYMBOLS}}` | 스크립팅 Define Symbols |
| `{{CODING_GUIDELINES}}` | 프로젝트 기반 코딩 체크리스트 |

저장 경로: `<프로젝트루트>/Docs/UNITY_DEV_ENV.md` (`Docs` 폴더 없으면 자동 생성)  
Claude.ai에서 직접 쓰기 불가 시 → `/mnt/user-data/outputs/Docs/UNITY_DEV_ENV.md`에 저장 후 경로 안내.

---

### 8단계: CLAUDE.md 업데이트

분석 완료 후 프로젝트 루트의 `CLAUDE.md`에 링크를 추가한다.

#### CLAUDE.md가 이미 존재하는 경우
- `## 개발 환경` 섹션 없으면 → 파일 끝에 추가
- 섹션 이미 있으면 → 링크만 최신화 (중복 추가 금지)

추가할 내용:
```markdown
---

## 개발 환경

> Unity 프로젝트 개발 환경 정보는 아래 파일을 참고하세요.  
> 코드 작성 전 Unity 버전, C# 기능, 패키지 목록을 확인할 수 있습니다.

- [UNITY_DEV_ENV.md](./Docs/UNITY_DEV_ENV.md) — Unity 버전, 렌더 파이프라인, 패키지, C# 사용 가능 기능 정리
```

#### CLAUDE.md가 없는 경우
프로젝트 루트에 새로 생성:
```markdown
# CLAUDE.md

이 파일은 Claude가 이 프로젝트를 이해하기 위한 참고 문서입니다.

---

## 개발 환경

> Unity 프로젝트 개발 환경 정보는 아래 파일을 참고하세요.  
> 코드 작성 전 Unity 버전, C# 기능, 패키지 목록을 확인할 수 있습니다.

- [UNITY_DEV_ENV.md](./Docs/UNITY_DEV_ENV.md) — Unity 버전, 렌더 파이프라인, 패키지, C# 사용 가능 기능 정리
```

---

### 9단계: 완료 보고

사용자에게 아래 내용을 요약 보고한다:
- Unity 버전 및 C# 버전
- 렌더 파이프라인
- 총 패키지 수 및 주요 패키지 목록
- 호환성 주의사항 (있을 경우)
- 저장된 파일 경로
