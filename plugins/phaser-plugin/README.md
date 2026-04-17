# Phaser Plugin

[![Version](https://img.shields.io/badge/Version-0.2.2-green.svg)]()

Phaser + Vite + Capacitor 환경에 특화된 스킬을 제공합니다. game-plugin의 범용 스킬과 함께 사용하여 Phaser 프로젝트의 Web/iOS/Android 빌드를 지원합니다.

---

## 스킬 (Skills)

| 스킬 | 설명 |
|------|------|
| [`phaser-build`](#phaser-build) | Web / iOS / Android 빌드 실행 (개발·배포, 서명 포함) |

---

#### `phaser-build`
Phaser + Vite + Capacitor 프로젝트의 빌드를 실행합니다.

- 빌드 요청 시 플랫폼·타입을 자동 파악, 불명확하면 질문
- **Web**: Vite dev server (개발) / Vite 프로덕션 빌드 (배포)
- **iOS**: xcodebuild CLI 자동화, 배포 시 환경 변수 기반 서명
- **Android**: Gradle 빌드, 배포 시 APK/AAB 선택 및 환경 변수 기반 키스토어 서명
- 환경 미충족(Phaser·Vite·Capacitor 누락) 시 빌드 중단 및 안내

---

## 플러그인 구조

```
phaser-plugin/
├── .claude-plugin/
│   └── plugin.json                  # 플러그인 매니페스트
└── skills/
    └── phaser-build/                # Web/iOS/Android 빌드
        └── SKILL.md
```
