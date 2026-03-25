---
name: project-analyst
description: 현재 프로젝트를 직접 탐색하여 패키지·라이브러리, 폴더 구조, 코딩 스타일·네이밍 규칙, 핵심 패턴, 테스트 구조, 개발 환경을 추출하고 Claude Code가 참조할 수 있는 PROJECT_CONTEXT.md 문서로 저장하는 스킬. 사용자가 "프로젝트 분석해줘", "아키텍처 파악해줘", "코딩 컨벤션 뽑아줘", "프로젝트 파악", "컨텍스트 문서 만들어줘", "이 프로젝트 어떻게 구성돼 있어?" 등의 표현을 사용하거나, 새 프로젝트에 처음 진입할 때 반드시 이 스킬을 사용하라. Unity 프로젝트를 우선 지원하며 범용 프로젝트도 분석 가능하다.
---

# Project Analyst

현재 작업 디렉토리의 프로젝트를 자동 탐색하여, Claude Code가 기능 구현 시 "이 프로젝트에서는 이렇게 한다"를 즉시 참조할 수 있는 `PROJECT_CONTEXT.md`를 생성하는 스킬.

## 워크플로우

### Step 1: 프로젝트 루트 탐색

먼저 프로젝트 유형을 판별한다.

```bash
# 현재 디렉토리 구조 파악 (2단계)
find . -maxdepth 2 -not -path '*/\.*' -not -path '*/node_modules/*' \
  -not -path '*/Library/*' -not -path '*/Temp/*' -not -path '*/obj/*' \
  -not -path '*/bin/*' | sort

# Unity 프로젝트 여부
ls ProjectSettings/ProjectVersion.txt 2>/dev/null

# 기타 프로젝트 판별
ls package.json tsconfig.json *.csproj *.sln go.mod requirements.txt 2>/dev/null
```

**Unity 프로젝트 확인 신호**: `ProjectSettings/`, `Assets/`, `Packages/manifest.json` 존재

### Step 2: 개발 환경 수집

#### Unity 프로젝트
```bash
# Unity 버전
cat ProjectSettings/ProjectVersion.txt

# 패키지 목록 (UPM)
cat Packages/manifest.json

# 스크립팅 백엔드, API 레벨 등
cat ProjectSettings/ProjectSettings.asset | grep -E "scriptingBackend|apiCompatibilityLevel|targetPlatform|companyName|productName"
```

#### 범용 프로젝트
```bash
# 언어/런타임 버전
cat package.json go.mod *.csproj requirements.txt Gemfile 2>/dev/null | head -50
```

### Step 3: 폴더 구조 & 파일 조직 분석

```bash
# Assets 하위 구조 (Unity)
find Assets -maxdepth 3 -type d -not -path '*/\.*' | sort

# 스크립트 최상위 구조
find Assets/Scripts -maxdepth 2 -type d 2>/dev/null | sort

# 어셈블리 정의 파일 위치 (레이어 경계 추론)
find Assets -name "*.asmdef" 2>/dev/null | sort

# 테스트 폴더 위치
find . -type d -name "Tests" -o -type d -name "Test" -o -type d -name "__tests__" \
  -o -type d -name "EditMode" -o -type d -name "PlayMode" 2>/dev/null | sort
```

### Step 4: 패키지 & 라이브러리 상세 분석

#### Unity
```bash
# manifest.json 전체 파싱
cat Packages/manifest.json

# 서드파티 패키지 (Assets 내 직접 설치된 것)
find Assets -name "package.json" -not -path "*/node_modules/*" 2>/dev/null \
  | xargs grep -l "name\|version" 2>/dev/null | head -20
```

핵심 패키지 식별 대상:
- DI 컨테이너: VContainer, Zenject, Pure DI
- 이벤트 버스: MessagePipe, UniRx, R3
- 비동기: UniTask, async/await
- 트위닝: DOTween, LeanTween
- 반응형: R3, UniRx
- 테스트: NUnit, Moq, NSubstitute

### Step 5: 실제 사용 패턴 추출

설치된 패키지가 **실제로 어떻게 사용되는지** 코드에서 찾아 예시를 추출한다.

```bash
# DI 패턴
grep -r "VContainer\|IContainerBuilder\|RegisterEntryPoint\|Zenject\|Bind" \
  Assets/Scripts --include="*.cs" -l | head -5

# 이벤트/메시지 패턴
grep -r "MessagePipe\|IPublisher\|ISubscriber\|Subject\|IObservable\|AddTo" \
  Assets/Scripts --include="*.cs" -l | head -5

# 비동기 패턴
grep -r "UniTask\|async\|await\|CancellationToken" \
  Assets/Scripts --include="*.cs" -l | head -5

# 반응형 패턴
grep -r "ReactiveProperty\|Observable\|Subscribe" \
  Assets/Scripts --include="*.cs" -l | head -5
```

발견된 파일을 1~2개 직접 읽어 **실제 사용 예시 코드**를 추출한다.

### Step 6: 코딩 스타일 & 네이밍 규칙 추출

대표 소스 파일 5~10개를 직접 읽어 패턴을 추출한다.

```bash
find Assets/Scripts -name "*.cs" | shuf | head -10
```

읽은 코드에서 추출할 항목:
- **네이밍**: 클래스명 규칙 (접두사/접미사 패턴), 인터페이스명 (`I` 접두사 여부), 메서드명 (동사형/명사형), 필드명 (`_camelCase` / `camelCase` / `PascalCase`), 이벤트명
- **코드 스타일**: 접근 제한자 순서, `var` 사용 여부, 람다 vs 메서드 선호, nullable 처리 방식
- **주석**: XML 문서 주석 사용 여부, 인라인 주석 스타일
- **파일 구조**: using 정렬, namespace 방식, partial class 사용 여부

### Step 7: 테스트 구조 분석

```bash
# 테스트 파일 목록
find . -name "*Tests.cs" -o -name "*Test.cs" | head -20

# EditMode vs PlayMode 구분
find . -path "*/EditMode/*" -name "*.cs" | head -5
find . -path "*/PlayMode/*" -name "*.cs" | head -5

# 테스트 대표 파일 읽기 (1~2개)
find . -name "*Tests.cs" | head -2 | xargs cat 2>/dev/null
```

추출 항목: 테스트 프레임워크, Mock 라이브러리, Arrange/Act/Assert 패턴, 테스트 클래스 네이밍, 테스트 메서드 네이밍

### Step 8: PROJECT_CONTEXT.md 작성

`references/output-template.md`를 읽은 후 수집된 정보로 채워 작성한다.

**작성 원칙**:
- 추상적 설명 금지 — 실제 코드 예시를 반드시 포함
- 설계 방향 제시 금지 — 현재 프로젝트에 있는 것만 기록
- 정보가 없는 항목은 "미확인 — 직접 확인 필요" 표기
- Claude Code가 이 문서만 보고 "이 프로젝트 방식대로" 코드를 작성할 수 있는 수준

### Step 9: 저장 & 전달

- 기본 저장 위치: 프로젝트 루트 `/PROJECT_CONTEXT.md`
- 출력 위치: `/mnt/user-data/outputs/PROJECT_CONTEXT.md` (다운로드용 사본)
- `present_files`로 사용자에게 전달

## Unity 특화 분석 힌트

| 패키지 감지 | 추론 패턴 |
|-------------|-----------|
| `com.cysharp.unitask` | 모든 비동기는 UniTask 사용 |
| `com.cysharp.r3` 또는 `com.neuecc.unirx` | 반응형 프로퍼티 / 스트림 패턴 |
| `com.hadashigames.vcontainer` | VContainer DI |
| `com.hadashigames.messagepipe` | MessagePipe 이벤트 버스 |
| `com.demigiant.dotween` | DOTween 애니메이션 |
| `nuget.moq` 또는 `nsubstitute` | Mock 프레임워크 |

어셈블리 정의(.asmdef) 파일 위치로 레이어 경계를 추론한다:
```bash
find Assets -name "*.asmdef" | xargs cat 2>/dev/null
```

## 품질 기준

생성된 `PROJECT_CONTEXT.md`는 다음을 충족해야 한다:
- [ ] 실제 코드 스니펫이 각 패턴마다 1개 이상 포함됨
- [ ] 네이밍 규칙이 예시와 함께 명시됨 (`PlayerPresenter` ✅ / `playerPresenter` ❌ 형태)
- [ ] DI 등록 방법이 실제 프로젝트 코드 기반으로 기술됨
- [ ] 새 파일 추가 시 어느 폴더에 어떤 파일을 만들어야 하는지 명확함
- [ ] 테스트 작성 방법이 실제 예시 기반으로 기술됨
- [ ] 설계 패턴을 강요하거나 현재 프로젝트에 없는 내용을 포함하지 않음
