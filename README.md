# darknaku-marketplace

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-purple.svg)](https://code.claude.com)
[![Version](https://img.shields.io/badge/Version-0.2.3-green.svg)]()
[![Author](https://img.shields.io/badge/Author-DarkNaku-orange.svg)](https://github.com/darknaku)

> **체계적인 게임 개발을 위한 Claude Code 플러그인 마켓플레이스**

darknaku-marketplace는 Claude Code용 플러그인 마켓플레이스입니다. 게임 개발의 전체 사이클을 AI와 함께 체계적으로 진행할 수 있도록 설계된 플러그인들을 제공합니다.

---

## 플러그인 목록

| 플러그인 | 버전 | 설명 |
|---------|:----:|------|
| **[Game Plugin](plugins/game-plugin/README.md)** | 0.2.2 | 엔진 독립적인 게임 개발 플러그인 — 기획, 분석, 설계, 구현, 커밋, 버그 수정 |
| **[Unity Plugin](plugins/unity-plugin/README.md)** | 0.2.3 | Unity + C# 환경에 특화된 환경 분석, 테스트, UI 구현 |
| **[Phaser Plugin](plugins/phaser-plugin/README.md)** | 0.2.2 | Phaser + Vite + Capacitor 환경에 특화된 빌드 (Web/iOS/Android) |

---

## 설치

### 마켓플레이스 추가

```bash
/plugin marketplace add darknaku/darknaku-marketplace
```

### 플러그인 설치

```bash
/plugin install game-plugin
/plugin install unity-plugin
/plugin install phaser-plugin
```

---

## 라이선스

MIT License. 자세한 내용은 [LICENSE](LICENSE)를 참조하세요.

---

Made by [DarkNaku](https://github.com/darknaku)
