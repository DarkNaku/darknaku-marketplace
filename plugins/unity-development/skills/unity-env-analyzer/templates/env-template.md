# Unity 개발 환경 정의서

> 이 파일은 Unity 프로젝트 분석을 통해 자동 생성되었습니다.  
> 코드 작성 시 이 파일을 참고하여 버전 호환성과 사용 가능한 API를 확인하세요.

---

## 1. Unity 환경

| 항목 | 값 |
|------|----|
| Unity 버전 | `{{UNITY_VERSION}}` |
| C# 버전 | {{CSHARP_VERSION}} |
| 스크립팅 백엔드 | {{SCRIPTING_BACKEND}} |
| API 호환성 레벨 | {{API_COMPATIBILITY}} |
| 렌더 파이프라인 | {{RENDER_PIPELINE}} |

---

## 2. 사용 가능한 C# 기능

이 Unity 버전({{CSHARP_VERSION}})에서 사용할 수 있는 주요 언어 기능:

### ✅ 사용 가능

{{CSHARP_AVAILABLE_FEATURES}}

### ❌ 사용 불가 (C# 10+, Unity 미지원)

- File-scoped namespaces (`namespace Foo;`)
- Global usings (`global using`)
- `record struct`
- List patterns (`[1, 2, ..]`)
- Required members (`required`)
- Primary constructors (클래스용)
- Collection expressions (`[1, 2, 3]`)

---

## 3. 렌더 파이프라인

**{{RENDER_PIPELINE_DETAIL}}**

{{RENDER_PIPELINE_NOTES}}

---

## 4. 설치된 패키지

{{PACKAGES_TABLE}}

---

## 5. NuGet 패키지

{{NUGET_SECTION}}

---

## 6. 어셈블리 구조

{{ASMDEF_TABLE}}

---

## 7. 빌드 타겟 및 플랫폼

| 항목 | 값 |
|------|----|
| 빌드 타겟 | {{BUILD_TARGET}} |
| 스크립팅 백엔드 | {{SCRIPTING_BACKEND}} |
| API 수준 | {{API_COMPATIBILITY}} |
| Define Symbols | {{DEFINE_SYMBOLS}} |

---

## 8. 코드 작성 가이드라인 요약

이 프로젝트에서 코드 작성 시 핵심 체크리스트:

{{CODING_GUIDELINES}}
