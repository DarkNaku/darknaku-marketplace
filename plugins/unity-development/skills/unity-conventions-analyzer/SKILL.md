---
name: unity-conventions-analyzer
description: Unity 프로젝트의 Scripts, Scenes, Prefabs, Resources 폴더를 직접 분석하여 코드 아키텍처, 디자인 패턴, 네임스페이스 관습, 암묵적 코딩 규칙을 역추론하고 UNITY_PROJECT_CONVENTIONS.md를 생성하는 스킬. unity-env-analyzer(환경/패키지 분석)와는 독립적으로 동작하며, 새로운 기능 개발 시 기존 프로젝트 스타일에 자연스럽게 녹아들 수 있도록 코드베이스의 관습을 문서화한다. 사용자가 "프로젝트 구조 파악", "코딩 컨벤션 정리", "아키텍처 분석", "기존 패턴 파악", "어떻게 코드 짜고 있어?", "스타일 맞춰줘", "기존 코드 스타일 분석해줘", "새 기능 추가 전에 코드 구조 먼저 봐줘" 등을 말할 때 반드시 이 스킬을 사용할 것.
---

# Unity 프로젝트 아키텍처 분석기

Unity 프로젝트의 **코드베이스 자체**를 읽어 아키텍처 패턴, 네이밍 관습, 암묵적 규칙을 역추론하고, 신규 기능 개발 시 참조할 수 있는 `UNITY_PROJECT_CONVENTIONS.md`를 생성한다.

> **unity-env-analyzer와의 구분**  
> - `unity-env-analyzer` → Unity 버전, 패키지, C# 기능 가용성 (환경)  
> - `unity-conventions-analyzer` → 폴더 구조, 디자인 패턴, 네이밍 관습, 암묵적 규칙 (코드 관습)  
> 두 스킬은 독립적으로 실행 가능하며, 함께 사용하면 더욱 완전한 프로젝트 이해가 가능하다.

---

## 0단계: Unity 프로젝트 여부 확인 (필수)

**이 스킬은 Unity 프로젝트에서만 동작한다.** 분석 시작 전 반드시 확인한다.

```bash
find . -maxdepth 4 \( \
  -name "ProjectVersion.txt" -path "*/ProjectSettings/*" -o \
  -name "manifest.json" -path "*/Packages/*" \
\) 2>/dev/null | head -5
```

| 판별 기준 | 판정 |
|-----------|------|
| `ProjectSettings/ProjectVersion.txt` 존재 | ✅ Unity 프로젝트 |
| `Packages/manifest.json` 존재 | ✅ Unity 프로젝트 |
| 둘 다 없음 | ❌ 즉시 중단 |

Unity 프로젝트가 아닌 경우 아래 메시지를 출력하고 **모든 분석을 중단**한다:

> "Unity 프로젝트를 찾을 수 없습니다. `ProjectSettings/ProjectVersion.txt` 또는 `Packages/manifest.json` 파일이 있는 Unity 프로젝트 폴더에서 실행해 주세요."

---

## 1단계: 프로젝트 루트 및 코드 경로 동적 탐색

스크립트 경로를 하드코딩하지 않는다. 아래 순서로 실제 구조를 먼저 파악한다.

### 1-1. Assets 폴더 위치 확인

```bash
# Assets 폴더 위치 탐색
find . -maxdepth 3 -type d -name "Assets" 2>/dev/null | head -3
```

### 1-2. .cs 파일 밀집 폴더 자동 발견

```bash
# Assets 하위에서 .cs 파일이 가장 많이 모여 있는 상위 폴더 탐색
find . -path "*/Assets/*" -name "*.cs" 2>/dev/null \
  | sed 's|/[^/]*\.cs$||' \
  | awk -F'/' '{print $1"/"$2"/"$3"/"$4}' \
  | sort | uniq -c | sort -rn | head -10
```

이 결과로 프로젝트의 **실제 코드 루트**를 결정한다:

| 탐지 예시 | 코드 루트 |
|-----------|-----------|
| `Assets/Scripts/` 에 집중 | `Assets/Scripts/` |
| `Assets/_Project/Scripts/` 에 집중 | `Assets/_Project/` |
| `Assets/MyGame/` 에 집중 | `Assets/MyGame/` |
| `Assets/Sources/` 에 집중 | `Assets/Sources/` |
| 여러 폴더에 분산 | 각 폴더 모두 분석 대상 |

### 1-3. 서드파티/템플릿 코드 분류

애매한 폴더를 **세 가지 카테고리**로 분류한다.

---

#### ① 자동 제외 — 사용자에게 묻지 않음

폴더명이 명확히 서드파티를 나타내는 경우:

```bash
find . -path "*/Assets/*" -type d \( \
  -name "Plugins" -o -name "ThirdParty" -o -name "Third_Party" \
  -o -name "AssetStore" -o -name "External" -o -name "Vendor" \
  -o -name "PackageCache" \
\) 2>/dev/null
```

---

#### ② 템플릿 추정 → 사용자에게 확인

아래 신호 중 2개 이상이면 **베이스 템플릿**으로 추정한다.  
템플릿 코드는 제외하지 않고 오히려 **분석의 기반 스타일**로 우선 취급한다.

| 템플릿 신호 | 탐지 방법 |
|------------|-----------|
| `abstract class` 또는 `virtual` 메서드가 많음 | "상속 의도로 설계된 코드" |
| 예제 Scene이 폴더 내에 포함 (`Example`, `Demo`, `Sample`) | `find [폴더] -name "*.unity"` |
| 주석에 `// Override`, `// Extend`, `// Customize` 류 존재 | grep으로 확인 |
| 폴더 구조가 게임 전체 흐름을 담고 있음 (`UI/`, `Gameplay/`, `Core/` 등) | 폴더 트리 확인 |
| `README`에 "이 코드를 수정/확장하세요" 류의 안내 | README 내용 확인 |

```bash
# 템플릿 후보: .unity 파일을 포함하는 Assets 직계 하위 폴더 탐색
find . -path "*/Assets/*/*" -name "*.unity" 2>/dev/null \
  | sed 's|/[^/]*$||' | sed 's|/[^/]*$||' | sort | uniq
```

템플릿으로 추정되는 폴더가 발견되면 사용자에게 확인한다:

```
📦 베이스 템플릿으로 추정되는 폴더가 발견되었습니다:

  - Assets/StarterAssets/   (예제 Scene 포함, abstract class 다수, 상속 구조)
  - Assets/GameTemplate/    (UI·Gameplay·Core 폴더 구조 포함, README에 확장 안내)

이 폴더가 프로젝트의 베이스 템플릿인가요?

  [예] → 이 폴더의 코딩 스타일을 기반 관습으로 분석합니다.
         새 기능도 이 스타일을 따르도록 가이드에 반영됩니다.
  [아니오] → 일반 서드파티로 처리합니다. (분석에서 제외할지 별도 확인)
```

---

#### ③ 유틸리티 플러그인 추정 → 제외 여부 확인

아래 신호 중 2개 이상이면 **유틸리티 플러그인**으로 추정한다:

| 플러그인 신호 | 탐지 방법 |
|-------------|-----------|
| `LICENSE`, `CHANGELOG` 파일 포함 | 배포 패키지 징후 |
| 네임스페이스가 프로젝트 루트와 완전히 다름 | 2단계 분석 후 교차 확인 |
| `.dll` 파일 또는 `[assembly: AssemblyVersion]` 포함 | 외부 배포 패키지 |
| 폴더명이 제품/회사명 형태 (ex: `DOTween`, `Sirenix`) | 프로젝트 네임스페이스와 무관 |

```bash
# 플러그인 후보 탐색
find . -path "*/Assets/*" -maxdepth 2 -type d 2>/dev/null \
  | grep -v -E "(Plugins|ThirdParty|Third_Party|AssetStore|External|Vendor|PackageCache)" \
  | xargs -I{} sh -c \
    'ls "{}" 2>/dev/null | grep -qiE "^(LICENSE|CHANGELOG)" && echo "PLUGIN_SUSPECT: {}"'
```

플러그인으로 추정되는 폴더가 발견되면 사용자에게 확인한다:

```
⚠️ 서드파티 플러그인으로 추정되는 폴더가 발견되었습니다:

  - Assets/DOTween/         (LICENSE 파일 포함, 네임스페이스 'DG.Tweening')
  - Assets/Odin Inspector/ (네임스페이스 'Sirenix'가 프로젝트와 무관)

이 폴더들을 분석에서 제외할까요?
제외하면 해당 폴더의 패턴은 관습 추론에 포함되지 않습니다.
```

---

#### 최종 분류 확정

사용자 응답을 반영해 세 버킷을 확정한 뒤 분석을 재개한다:

| 버킷 | 처리 방식 |
|------|-----------|
| **자동 제외** | 모든 분석에서 완전 제외 |
| **베이스 템플릿** | 분석 포함 + 문서에 "기반 스타일 출처" 명시 |
| **유틸리티 플러그인** | 사용자 선택에 따라 제외 또는 포함 |

의심 폴더가 하나도 없으면 이 단계 전체를 건너뛰고 바로 진행한다.

`.cs` 파일이 전혀 없으면:
> "분석할 C# 스크립트를 찾을 수 없습니다. Unity 프로젝트 폴더 또는 Assets 폴더가 포함된 경로에서 실행해 주세요."

---

## 2단계: 폴더 구조 스캔

0단계에서 확정한 **코드 루트 경로**를 `$CODE_ROOT`로 사용한다.

```bash
# 코드 루트 하위 전체 폴더 트리 (최대 4단계)
find "$CODE_ROOT" -type d -not -path "*/.*" 2>/dev/null | sort

# Scenes 폴더 (Assets 전체에서 탐색)
find . -path "*/Assets/*" -name "*.unity" 2>/dev/null | sed 's|/[^/]*$||' | sort | uniq

# Prefabs 폴더 (Assets 전체에서 탐색)
find . -path "*/Assets/*" -name "*.prefab" 2>/dev/null | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -15

# Resources 폴더 유무
find . -path "*/Assets/*" -type d -name "Resources" 2>/dev/null

# .asmdef 파일 목록
find . -name "*.asmdef" 2>/dev/null
```

폴더 구조로부터 추론:
- 레이어 분리 방식 (예: `UI/`, `Gameplay/`, `Core/`, `Data/` 등)
- 기능별 vs 레이어별 분리 여부
- Editor 전용 폴더 유무
- `_Project`, `_Game` 같은 언더스코어 접두사 관습 여부

---

## 3단계: 네임스페이스 및 클래스 이름 패턴 추출

### 3-1. 네임스페이스 관습

```bash
# 코드 루트($CODE_ROOT, 0단계에서 확정) 내 .cs 파일에서 namespace 선언 추출
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | head -50 \
  | xargs grep -h "^namespace " 2>/dev/null | sort | uniq -c | sort -rn | head -20
```

분석 포인트:
- 공통 루트 네임스페이스 (예: `MyGame`, `Studio.ProjectName`)
- 네임스페이스 계층 깊이
- 폴더 구조와 네임스페이스가 일치하는지 여부
- 네임스페이스를 아예 사용하지 않는 프로젝트인지

### 3-2. 클래스 역할 접미사 패턴

```bash
# 클래스 이름 접미사 통계
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -h "^public class\|^public abstract class\|^public sealed class" 2>/dev/null \
  | grep -oP '(?<=class )\w+' | grep -oP '(Manager|Controller|System|Handler|Service|Factory|Repository|Helper|Base|Util|Component|Behaviour|Behavior|View|Model|Presenter|State|Command|Observer|Mediator|Installer|Provider|Loader|Spawner|Pool)$' \
  | sort | uniq -c | sort -rn
```

발견된 접미사를 바탕으로 프로젝트가 사용하는 역할 분리 방식을 서술한다.

---

## 4단계: 디자인 패턴 탐지

아래 패턴을 코드에서 실제로 탐지한다. 탐지된 패턴만 문서에 포함한다.

### Singleton

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -l "Instance\|_instance\|Singleton" 2>/dev/null | head -10
# 대표 파일 내용 확인
```

탐지 기준: `static.*Instance`, `private static.*_instance`, `DontDestroyOnLoad` 조합

### Observer / Event

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "UnityEvent|event Action|event Func|\.AddListener|\.RemoveListener|EventHandler" 2>/dev/null | head -10
```

탐지 기준: `UnityEvent`, C# `event Action`, 커스텀 이벤트 버스 클래스

### Object Pool

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "ObjectPool|Pool<|Dequeue|Enqueue|pooled" 2>/dev/null | head -10
```

### State Machine

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "IState|StateMachine|currentState|ChangeState|Transition" 2>/dev/null | head -10
```

### Command / Strategy / Factory

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "ICommand|Execute\(\)|IStrategy|Factory\.Create|CreateInstance" 2>/dev/null | head -10
```

### ScriptableObject 기반 데이터

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "ScriptableObject|CreateAssetMenu" 2>/dev/null | wc -l
# 수가 많으면 SO 기반 설계 채택으로 판단
```

### Dependency Injection (Zenject/VContainer)

```bash
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "Inject\b|\[Inject\]|IInstaller|LifetimeScope|Container\.Bind" 2>/dev/null | head -5
```

---

## 5단계: 리소스 로딩 방식 판별

```bash
# Addressables 사용 여부
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "Addressables\.|AssetReference|LoadAssetAsync" 2>/dev/null | head -5

# Resources.Load 사용 여부
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "Resources\.Load|Resources\.LoadAsync" 2>/dev/null | head -5

# AssetBundle 사용 여부
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | xargs grep -lE "AssetBundle|LoadFromFile|LoadFromMemory" 2>/dev/null | head -5
```

판별 결과:
| 방식 | 탐지 여부 | 비고 |
|------|-----------|------|
| Addressables | O/X | 주 방식인지 혼용인지 |
| Resources.Load | O/X | 레거시 잔존 여부 |
| AssetBundle | O/X | 커스텀 번들링 여부 |

---

## 6단계: Scene 및 Prefab 조직 방식 분석

```bash
# Scene 파일 목록
find . -path "*/Assets/*" -name "*.unity" 2>/dev/null | sort

# Prefab 파일 목록 (폴더별 집계)
find . -path "*/Assets/*" -name "*.prefab" 2>/dev/null | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -20
```

분석 포인트:
- Scene 명명 규칙 (예: `00_Boot`, `01_Login`, `Game_Level_01`)
- Scene 역할 분리 (Boot / UI / Gameplay 구분 여부)
- Prefab 폴더 조직 (기능별 / 타입별 / 레이어별)
- Prefab 명명 접두사/접미사 패턴

---

## 7단계: 암묵적 규칙 역추론

아래 항목을 실제 코드 샘플에서 확인하여 **명시되지 않았지만 반복적으로 쓰이는 패턴**을 기록한다.

### 7-1. 코드 샘플링

```bash
# 가장 많이 참조되거나 핵심으로 보이는 파일 5~10개 선택하여 읽기
find "$CODE_ROOT" -name "*.cs" 2>/dev/null | head -20
# → 파일 크기, 이름, 위치로 핵심 파일 선별 후 view로 내용 확인
```

### 7-2. 체크리스트

실제 코드에서 아래 항목을 확인한다:

| 항목 | 확인 방법 | 문서화 내용 |
|------|-----------|-------------|
| 직렬화 방식 | `[SerializeField]` vs `public` 필드 | 어느 쪽을 주로 사용하는지 |
| Awake vs Start | 초기화 메서드 선호도 | Awake/Start 역할 분리 관습 |
| Update 분기 | if/switch 분기 vs State 패턴 | 복잡한 Update 로직 처리 방식 |
| 코루틴 사용 | `StartCoroutine` vs `async/await` | 비동기 처리 방식 선호 |
| 주석 스타일 | XML doc (`///`) vs `//` 인라인 | 주석 문화 |
| 상수 관리 | `const`, `static readonly`, SO, `enum` | 상수/설정값 관리 방식 |
| 접근 제한자 | `private`/`protected` 사용 엄격도 | 캡슐화 수준 |

---

## 8단계: 스타일 가이드 충돌 감지 및 분류

분석 결과를 `UNITY_CODING_STANDARDS.md` / `MODERN_CSHARP_STYLE.md` 등 프로젝트에 존재하는 스타일 가이드와 대조한다.

```bash
# 프로젝트 내 스타일 가이드 파일 탐색
find . -maxdepth 3 \( \
  -name "CODING_STANDARDS*" -o -name "STYLE_GUIDE*" \
  -o -name "*CODING_STYLE*" -o -name "MODERN_CSHARP*" \
  -o -name "UNITY_CODING*" \
\) 2>/dev/null
```

스타일 가이드가 없으면 이 단계를 건너뛴다.

---

### 충돌 분류 원칙

충돌 항목은 그 성격에 따라 두 가지로 자동 분류한다. 사용자에게 별도로 묻지 않는다.

#### 🔵 코드 스타일 충돌 → 스타일 가이드 우선

프로젝트 관습은 **레거시(Legacy)**로 표시한다. 신규 코드는 스타일 가이드를 따른다.

| 해당 항목 |
|-----------|
| 직렬화 방식 (`public` 필드 vs `[SerializeField] private`) |
| 비동기 처리 방식 (Coroutine vs `async/await`) |
| 네이밍 케이스 (`camelCase` vs `PascalCase` 등) |
| 접근 제한자 엄격도 |
| 주석 스타일 |
| C# 언어 기능 사용 수준 (스타일 가이드의 버전 경계 기준 적용) |

**문서 표기 예시:**
```
| 직렬화 | ~~public 필드~~ → `[SerializeField] private` (스타일 가이드 기준 적용, 기존 코드는 레거시) |
```

#### 🟠 아키텍처 충돌 → 프로젝트 관습 우선

스타일 가이드 권장사항이 있더라도 **프로젝트에 이미 정착된 구조를 따른다**.

| 해당 항목 |
|-----------|
| 디자인 패턴 선택 (Singleton 구현 방식, 이벤트 버스 구조 등) |
| 폴더 및 어셈블리 구조 |
| 클래스 역할 분리 방식 (Manager / System / Controller 구분) |
| 리소스 로딩 전략 (단, 레거시 방식이 명확하면 코드 스타일로 재분류) |
| Scene / Prefab 조직 방식 |

**문서 표기 예시:**
```
| Singleton 구현 | static Instance + DontDestroyOnLoad (프로젝트 관습 유지) |
```

---

### 충돌 목록 정리

감지된 충돌 항목을 아래 형식으로 정리한다. 문서의 `## 9. 스타일 가이드와의 충돌` 섹션에 삽입된다.

| 항목 | 프로젝트 관습 | 스타일 가이드 | 적용 원칙 | 신규 코드 지침 |
|------|--------------|--------------|-----------|--------------|
| 직렬화 | `public` 필드 | `[SerializeField] private` | 🔵 스타일 가이드 우선 | `[SerializeField] private` 사용 |
| 이벤트 버스 | 커스텀 `EventManager` 클래스 | `event Action` 권장 | 🟠 프로젝트 관습 유지 | 기존 `EventManager` 패턴 따름 |

---

## 9단계: UNITY_PROJECT_CONVENTIONS.md 생성

출력 문서 구조는 `templates/conventions-template.md`를 참고한다.

**탐지되지 않은 항목은 "확인되지 않음"으로 채우지 않고 해당 섹션 자체를 생략한다.**

템플릿의 `[placeholder]` 표기는 모두 실제 분석값으로 교체한다.

---

## 10단계: CLAUDE.md 업데이트

분석 완료 후 프로젝트 루트의 `CLAUDE.md`에 링크를 추가한다.

**CLAUDE.md가 있는 경우** → `## 코드 관습` 섹션이 없으면 추가, 있으면 링크만 최신화:

```markdown
## 코드 관습

> 프로젝트 아키텍처, 디자인 패턴, 명명 규칙 등 코드 관습은 아래 파일을 참고하세요.

- [UNITY_PROJECT_CONVENTIONS.md](./Docs/UNITY_PROJECT_CONVENTIONS.md) — 폴더 구조, 디자인 패턴, 네이밍 관습, 암묵적 규칙
```

**CLAUDE.md가 없는 경우** → 새로 생성:

```markdown
# CLAUDE.md

이 파일은 Claude가 이 프로젝트를 이해하기 위한 참고 문서입니다.

---

## 코드 관습

> 프로젝트 아키텍처, 디자인 패턴, 명명 규칙 등 코드 관습은 아래 파일을 참고하세요.

- [UNITY_PROJECT_CONVENTIONS.md](./Docs/UNITY_PROJECT_CONVENTIONS.md) — 폴더 구조, 디자인 패턴, 네이밍 관습, 암묵적 규칙
```

---

## 11단계: 파일 저장 및 요약 보고

저장 위치: 프로젝트 루트의 `Docs/UNITY_PROJECT_CONVENTIONS.md`

Claude.ai에서 직접 저장 불가 시: `/mnt/user-data/outputs/Docs/UNITY_PROJECT_CONVENTIONS.md`

완료 후 사용자에게 보고:
- 분석된 C# 파일 수
- 탐지된 디자인 패턴 목록
- 리소스 로딩 방식
- 발견된 주요 암묵적 규칙 요약 (3~5개)
- 주의사항 (관습 불일치, 레거시 코드 혼재 등)

---

## 주의사항

- 코드 샘플은 **실제로 파일을 읽어** 패턴을 확인한다. 파일명만으로 추측하지 않는다.
- 탐지되지 않은 패턴은 "미사용"으로 단정하지 않고 "확인되지 않음"으로 표기하거나 섹션을 생략한다.
- 같은 패턴이 여러 방식으로 혼재할 경우(예: UnityEvent와 event Action 혼용) 두 방식을 모두 기록하고 더 많이 쓰이는 쪽을 "주 방식"으로 표기한다.
- Asset Store 구매 에셋 내부 코드는 관습 분석 대상에서 제외한다 (`/Assets/ThirdParty/`, `/Assets/Plugins/` 하위).
- 분석 결과는 현 시점 스냅샷이다. 프로젝트 구조가 크게 변경되면 재실행을 권장한다.
