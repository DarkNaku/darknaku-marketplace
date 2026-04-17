---
name: game-env-analyzer
description: 프로젝트 폴더를 분석하여 개발 환경 문서(DEV_ENV.md)를 생성하는 범용 스킬. Unity, Unreal, Godot, Phaser, Love2D, Cocos2d 등 게임 엔진과 Node.js, Python, Rust, Go 등 일반 개발 환경을 자동 감지한다. 전용 분석 스킬이 있는 엔진이 감지되면 해당 스킬을 자동으로 이어서 실행한다. 사용자가 "프로젝트 분석", "개발 환경 파악", "패키지 정리", "환경 문서화", "어떤 패키지 쓰고 있어?", "환경 파일 만들어줘" 등을 말할 때 반드시 이 스킬을 사용할 것.
---

# 개발 환경 분석기

프로젝트 폴더를 분석하여 개발 환경 문서를 생성한다.  
프로젝트 타입을 자동 감지하고, 전용 분석 스킬이 있으면 자동으로 이어서 실행한다.

> **출력 파일명**: 환경에 관계없이 항상 `Docs/DEV_ENV.md`로 저장한다.  
> **출력 템플릿**: `templates/game-env-template.md` 참고 (전용 스킬 없는 경우)

---

## 분석 순서

### 1단계: 프로젝트 타입 감지

아래 마커를 탐색하여 프로젝트 타입을 판별한다. 복수 타입이 감지될 수 있다.

```bash
find . -maxdepth 4 \( \
  -name "ProjectVersion.txt" \
  -o -name "manifest.json" -path "*/Packages/*" \
  -o -name "*.uproject" \
  -o -name "project.godot" \
  -o -name "conf.lua" \
  -o -name "cocos2d.js" -o -name "project.json" -path "*/cocos*" \
  -o -name "package.json" \
  -o -name "requirements.txt" -o -name "Pipfile" -o -name "pyproject.toml" \
  -o -name "*.csproj" -o -name "*.sln" \
  -o -name "Cargo.toml" \
  -o -name "go.mod" \
\) 2>/dev/null
```

#### 게임 엔진 마커

| 마커 파일 / 조건 | 엔진 |
|-----------------|------|
| `ProjectSettings/ProjectVersion.txt` 또는 `Packages/manifest.json` | **Unity** |
| `*.uproject` | **Unreal Engine** |
| `project.godot` | **Godot** |
| `conf.lua` | **Love2D** |
| `cocos2d.js` 또는 `project.json` (Cocos 경로) | **Cocos2d** |
| `package.json` + dependencies에 `phaser` 포함 | **Phaser** |
| `package.json` + `three` / `babylon` / `pixi` 포함 | **HTML5 WebGL** |
| `package.json` + `index.html` + 빌드 도구 없음 | **HTML5 Vanilla** |

#### 일반 개발 환경 마커

| 마커 파일 | 타입 |
|-----------|------|
| `package.json` (게임 엔진 외) | Node.js / JavaScript / TypeScript |
| `requirements.txt` / `Pipfile` / `pyproject.toml` | Python |
| `*.csproj` / `*.sln` (Unity 외) | .NET / C# |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| 감지 불가 | 사용자에게 타입 확인 요청 |

---

### 2단계: 전문 분석 스킬 확인 및 위임

감지된 엔진에 대응하는 전용 스킬이 **available_skills**에 있으면 즉시 이어서 실행한다.  
전용 스킬이 없으면 3단계 공통 분석으로 계속 진행한다.

| 감지된 엔진 | 전용 스킬 |
|------------|-----------|
| Unity | `unity-env-analyzer` |

> 전용 스킬 실행 시 해당 스킬의 SKILL.md를 읽고 절차를 따른다.  
> **출력 파일명은 전용 스킬과 무관하게 `Docs/DEV_ENV.md`로 통일한다.**

---

### 3단계: 런타임 / 언어 버전 추출

프로젝트 타입별로 아래 파일에서 **실제 사용 버전**을 추출한다.

#### Node.js
```bash
cat .nvmrc 2>/dev/null                          # 우선순위 1
cat .node-version 2>/dev/null                   # 우선순위 2
node -v 2>/dev/null                             # 시스템 버전 fallback
```
`package.json`의 `engines.node` 필드도 확인한다:
```json
{ "engines": { "node": ">=18.0.0" } }
```

#### Python
```bash
cat .python-version 2>/dev/null                 # pyenv 버전 파일
cat runtime.txt 2>/dev/null                     # Heroku 등
python3 --version 2>/dev/null                   # 시스템 버전 fallback
```
`pyproject.toml`의 `requires-python` 필드도 확인한다:
```toml
[project]
requires-python = ">=3.11"
```

#### Rust
```bash
cat rust-toolchain.toml 2>/dev/null             # 툴체인 파일
cat rust-toolchain 2>/dev/null                  # 구형 형식
```
`Cargo.toml`의 `edition` 필드:
```toml
[package]
edition = "2021"   ← 2015 / 2018 / 2021
```

#### Go
`go.mod`의 `go` 지시문:
```
module myapp
go 1.21         ← 이 줄 추출
```

#### .NET / C#
`*.csproj`의 `TargetFramework`:
```xml
<TargetFramework>net8.0</TargetFramework>
<LangVersion>12.0</LangVersion>
```

#### Unreal Engine
`.uproject`의 `EngineAssociation`:
```json
{ "EngineAssociation": "5.3" }
```

#### Godot
`project.godot`의 `config/features`:
```ini
[application]
config/features=PackedStringArray("4.2", "C#")  ← Godot 버전 + C# 사용 여부
```
C# 사용 여부는 `*.csproj` 존재 또는 `*.cs` 파일로도 확인한다.

#### Love2D
`conf.lua`의 `t.version`:
```lua
function love.conf(t)
    t.version = "11.4"   ← 이 줄 추출
end
```

#### Cocos2d
`project.json` 또는 `cocos2d.js`에서 버전 추출. Creator 프로젝트는 `package.json`의 `creator-version` 필드 확인:
```json
{ "creator-version": "3.7.0" }
```

---

### 4단계: 패키지 / 의존성 추출

**lock 파일을 우선 사용한다.** lock 파일이 없으면 선언 파일을 사용하되, 문서에 "(선언 버전, lock 파일 없음)"으로 표기한다.

#### Node.js
```bash
# lock 파일 우선순위: package-lock.json > yarn.lock > pnpm-lock.yaml
cat package-lock.json 2>/dev/null   # npm
cat yarn.lock 2>/dev/null           # yarn
cat pnpm-lock.yaml 2>/dev/null      # pnpm
```
`package.json`에서 `dependencies`와 `devDependencies`를 구분하여 추출한다.

#### Python
```bash
# lock 파일 우선순위: poetry.lock > Pipfile.lock > requirements.txt
cat poetry.lock 2>/dev/null
cat Pipfile.lock 2>/dev/null
cat requirements.txt 2>/dev/null
```
`pyproject.toml`의 `[project.dependencies]` 또는 `[tool.poetry.dependencies]`도 읽는다.

#### Rust
```bash
cat Cargo.lock 2>/dev/null          # 실제 설치 버전
cat Cargo.toml 2>/dev/null          # 선언 버전 (workspace 포함)
```

#### Go
```bash
cat go.sum 2>/dev/null              # 실제 설치 버전 해시
cat go.mod 2>/dev/null              # require 블록에서 의존성 추출
```

#### .NET / C#
```bash
find . -name "*.csproj" 2>/dev/null  # PackageReference 목록
```
```xml
<PackageReference Include="Newtonsoft.Json" Version="13.0.1" />
```

#### Unreal Engine
```bash
# 플러그인 목록
cat *.uproject 2>/dev/null          # Plugins 배열
find . -name "*.uplugin" 2>/dev/null  # 커스텀 플러그인
find . -path "*/ThirdParty/*" -name "*.build.cs" 2>/dev/null  # 서드파티 라이브러리
```
`.uproject`의 Plugins 배열:
```json
{
  "Plugins": [
    { "Name": "EnhancedInput", "Enabled": true },
    { "Name": "Niagara", "Enabled": true }
  ]
}
```

#### Godot
```bash
find . -name "*.gdextension" 2>/dev/null   # GDExtension 플러그인
ls addons/ 2>/dev/null                     # 설치된 애드온 목록
```
각 애드온의 `plugin.cfg`에서 버전 확인:
```ini
[plugin]
name="DialogueManager"
version="2.10.0"
```

#### Love2D
Love2D는 표준 패키지 매니저가 없다. 아래 방법으로 외부 라이브러리를 탐지한다:
```bash
# 주요 Lua 라이브러리 폴더/파일명으로 탐지
find . -maxdepth 3 \( \
  -name "middleclass.lua" \
  -o -name "hump" -type d \
  -o -name "flux.lua" \
  -o -name "bump.lua" \
  -o -name "anim8.lua" \
  -o -name "lurker.lua" \
  -o -name "lovebird.lua" \
\) 2>/dev/null
```

#### Cocos2d
```bash
cat frameworks/package.json 2>/dev/null    # Cocos Creator
find . -name "*.json" -path "*/packages/*" 2>/dev/null  # 설치된 패키지
```

#### Phaser / HTML5
```bash
cat package-lock.json 2>/dev/null   # 실제 설치 버전 우선
cat package.json 2>/dev/null        # dependencies + devDependencies
```
CDN 사용 시 `index.html`의 `<script>` 태그에서 버전 추출:
```html
<script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
```

---

### 5단계: 문서 생성 및 저장

`templates/game-env-template.md`를 읽고 플레이스홀더를 실제 분석값으로 채워 저장한다.

**모든 패키지 테이블에는 반드시 버전을 기재한다.** 버전을 확인할 수 없는 경우 "버전 미확인"으로 표기하고 확인 방법을 안내한다.

저장 경로: `<프로젝트루트>/Docs/DEV_ENV.md` (`Docs` 폴더 없으면 자동 생성)  
Claude.ai에서 직접 쓰기 불가 시 → `/mnt/user-data/outputs/Docs/DEV_ENV.md`에 저장 후 경로 안내.

---

### 6단계: CLAUDE.md 업데이트

- `## 개발 환경` 섹션 없으면 → 파일 끝에 추가
- 섹션 이미 있으면 → 링크만 최신화 (중복 추가 금지)
- `CLAUDE.md` 없으면 → 새로 생성

추가 내용:
```markdown
---

## 개발 환경

- [DEV_ENV.md](./Docs/DEV_ENV.md) — 개발 환경 정보 (엔진/언어 버전, 패키지, 빌드 설정)
```

---

### 7단계: 완료 보고

- 감지된 프로젝트 타입
- 언어 / 런타임 버전 (버전 출처 명시: lock 파일 / 선언 파일 / 시스템)
- 패키지 총 수 및 주요 패키지 목록
- 버전 미확인 항목이 있으면 별도 언급
- 저장된 파일 경로
