---
paths:
  - "**/*.ts"
  - "**/*.js"
---

# Phaser 코딩 스타일 가이드

> **적용 범위**: 프로젝트 내 모든 `*.ts`, `*.js` 파일
> **참고**: [phaserjs/template-webpack-ts](https://github.com/phaserjs/template-webpack-ts), [phaserjs/template-webpack](https://github.com/phaserjs/template-webpack)

---

## 목차

1. [파일 & 폴더 구조](#1-파일--폴더-구조)
2. [명명 규칙](#2-명명-규칙)
3. [포매팅](#3-포매팅)
4. [임포트](#4-임포트)
5. [Scene 클래스 구조](#5-scene-클래스-구조)
6. [TypeScript 전용 규칙](#6-typescript-전용-규칙)
7. [에셋 로드](#7-에셋-로드)
8. [금지 사항](#8-금지-사항)
9. [예제 파일](#9-예제-파일)

---

## 1. 파일 & 폴더 구조

```
src/
├── main.ts (또는 main.js)      ← DOM 엔트리 포인트
├── global.d.ts                  ← 에셋 모듈 선언 (TypeScript)
└── game/
    ├── main.ts (또는 main.js)  ← Phaser Game 설정 및 생성
    └── scenes/                  ← Scene 클래스 파일
        ├── Boot.ts
        ├── Preloader.ts
        ├── MainMenu.ts
        ├── Game.ts
        └── GameOver.ts
```

- 파일 하나에 **클래스 하나**
- **파일명 == 클래스명** (PascalCase, 대소문자 완전 일치)
- Scene 클래스는 `src/game/scenes/` 디렉토리에 배치
- 정적 에셋은 `public/assets/` 하위에 유형별 분류 (`images/`, `audio/`, `tilemaps/`, `fonts/`)

---

## 2. 명명 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| 클래스 | PascalCase | `MainMenu`, `GameOver`, `Preloader` |
| Scene 키 (super 인자) | PascalCase 문자열 | `super('MainMenu')` |
| 메서드 | camelCase | `preload()`, `create()`, `update()` |
| 변수 / 프로퍼티 | camelCase | `background`, `logo`, `msgText` |
| 상수 | UPPER_SNAKE_CASE | `MAX_HEALTH`, `TILE_SIZE` |
| 함수 (export) | PascalCase | `StartGame` |
| 이벤트 키 | kebab-case 또는 camelCase | `'pointerdown'`, `'animationcomplete'` |

---

## 3. 포매팅

### 브레이스 스타일 — Allman

클래스 선언, 메서드 선언에서 여는 브레이스를 **다음 줄**에 배치한다:

```typescript
export class Boot extends Scene
{
    constructor ()
    {
        super('Boot');
    }

    preload ()
    {
        this.load.image('background', 'assets/bg.png');
    }

    create ()
    {
        this.scene.start('Preloader');
    }
}
```

### 콜백 / 인라인 — K&R

화살표 함수, 콜백, 설정 객체 등 인라인 블록은 같은 줄에 여는 브레이스를 배치한다:

```typescript
this.input.once('pointerdown', () => {

    this.scene.start('Game');

});

const config = {
    type: AUTO,
    width: 1024,
    height: 768,
};
```

### 기타

- 들여쓰기: **4스페이스**
- 따옴표: **작은따옴표** (`'`)
- 세미콜론: **사용**
- 콜백 본문 전후 **빈 줄 1개**
- 메서드 사이 **빈 줄 1개**

---

## 4. 임포트

### Phaser 임포트

Phaser에서 필요한 것만 named import로 가져온다:

```typescript
// ✅ named import
import { Scene } from 'phaser';
import { Scene, GameObjects } from 'phaser';
import { AUTO, Game } from 'phaser';

// ❌ 전체 import
import Phaser from 'phaser';
import * as Phaser from 'phaser';
```

단, 타입 참조가 필요한 경우 네임스페이스 접근은 허용한다:

```typescript
// ✅ 타입 참조 시 허용
const config: Phaser.Types.Core.GameConfig = { ... };
```

### Scene 임포트

Scene 클래스는 상대 경로로 가져온다:

```typescript
import { Boot } from './scenes/Boot';
import { Game as MainGame } from './scenes/Game';
```

Phaser의 `Game` 클래스와 이름이 충돌하는 Scene은 **별칭**을 사용한다:

```typescript
import { Game as MainGame } from './scenes/Game';
```

### 임포트 순서

1. 외부 패키지 (`phaser` 등)
2. 내부 모듈 (Scene 클래스 등)

```typescript
import { Boot } from './scenes/Boot';
import { Game as MainGame } from './scenes/Game';
import { GameOver } from './scenes/GameOver';
import { MainMenu } from './scenes/MainMenu';
import { AUTO, Game } from 'phaser';
import { Preloader } from './scenes/Preloader';
```

---

## 5. Scene 클래스 구조

### 기본 패턴

Scene 클래스는 `Phaser.Scene`을 상속하며, `constructor`에서 Scene 키를 전달한다:

```typescript
export class MainMenu extends Scene
{
    constructor ()
    {
        super('MainMenu');
    }
}
```

### 멤버 선언 순서

1. **프로퍼티 선언** (TypeScript에서 타입 명시)
2. **constructor** — `super(SceneKey)` 호출
3. **Phaser 라이프사이클 메서드** — `init` → `preload` → `create` → `update` 순서
4. **커스텀 메서드**

```typescript
export class Game extends Scene
{
    // 1. 프로퍼티
    camera: Phaser.Cameras.Scene2D.Camera;
    background: Phaser.GameObjects.Image;

    // 2. constructor
    constructor ()
    {
        super('Game');
    }

    // 3. 라이프사이클: create
    create ()
    {
        this.camera = this.cameras.main;
        this.camera.setBackgroundColor(0x00ff00);

        this.background = this.add.image(512, 384, 'background');
        this.background.setAlpha(0.5);
    }

    // 4. 커스텀 메서드
    // ...
}
```

### Scene 전환

`this.scene.start(SceneKey)` 로 전환한다. Scene 키는 문자열 리터럴을 사용한다:

```typescript
this.scene.start('MainMenu');
this.scene.start('Game');
this.scene.start('GameOver');
```

### 이벤트 바인딩

일회성 이벤트에는 `once`, 반복 이벤트에는 `on`을 사용한다:

```typescript
// 일회성
this.input.once('pointerdown', () => {

    this.scene.start('Game');

});

// 반복
this.input.on('pointermove', (pointer) => {

    // ...

});
```

---

## 6. TypeScript 전용 규칙

### 프로퍼티 타입 선언

Scene 프로퍼티는 Phaser 네임스페이스 타입 또는 named import 타입을 사용한다:

```typescript
// 네임스페이스 참조
camera: Phaser.Cameras.Scene2D.Camera;
background: Phaser.GameObjects.Image;

// named import 사용
import { Scene, GameObjects } from 'phaser';

background: GameObjects.Image;
logo: GameObjects.Image;
title: GameObjects.Text;
```

### Game Config 타입

```typescript
const config: Phaser.Types.Core.GameConfig = {
    type: AUTO,
    width: 1024,
    height: 768,
    parent: 'game-container',
    backgroundColor: '#028af8',
    scene: [Boot, Preloader, MainMenu, MainGame, GameOver]
};
```

### tsconfig.json 권장 설정

```json
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "ESNext",
        "moduleResolution": "bundler",
        "strict": true,
        "strictPropertyInitialization": false,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "noFallthroughCasesInSwitch": true,
        "isolatedModules": true,
        "resolveJsonModule": true,
        "skipLibCheck": true
    }
}
```

- `strictPropertyInitialization: false` — Phaser Scene 프로퍼티는 `create()`에서 초기화하므로 이 옵션을 끈다
- `strict: true` — 나머지 strict 옵션은 모두 활성화

### 에셋 모듈 선언

TypeScript 프로젝트에서는 `src/global.d.ts`에 에셋 모듈을 선언한다:

```typescript
declare module '*.png' {
    const src: string
    export default src
}
declare module '*.jpg' {
    const src: string
    export default src
}
// 필요한 확장자 추가
```

---

## 7. 에셋 로드

### Boot Scene에서 최소 에셋 로드

```typescript
export class Boot extends Scene
{
    preload ()
    {
        this.load.image('background', 'assets/bg.png');
    }

    create ()
    {
        this.scene.start('Preloader');
    }
}
```

### Preloader Scene에서 게임 에셋 로드

`setPath`로 에셋 경로를 설정한 뒤 개별 에셋을 로드한다:

```typescript
preload ()
{
    this.load.setPath('assets');

    this.load.image('logo', 'logo.png');
    this.load.image('player', 'player.png');
}
```

### 에셋 키

- camelCase 또는 kebab-case 문자열 사용
- 의미가 명확한 이름 사용 (`'bg'` 보다 `'background'`)

---

## 8. 금지 사항

- `var` 사용 금지 — `const` 또는 `let` 사용
- `any` 타입 남용 금지 (TypeScript) — 구체적 타입 또는 Phaser 제공 타입 사용
- 전역 변수 사용 금지 — Scene 프로퍼티 또는 모듈 스코프 사용
- 매직 넘버 금지 — 이름 있는 상수로 추출
- `console.log` 배포 코드에 남기기 금지

---

## 9. 예제 파일

### TypeScript Scene

```typescript
import { Scene, GameObjects } from 'phaser';

export class MainMenu extends Scene
{
    background: GameObjects.Image;
    logo: GameObjects.Image;
    title: GameObjects.Text;

    constructor ()
    {
        super('MainMenu');
    }

    create ()
    {
        this.background = this.add.image(512, 384, 'background');

        this.logo = this.add.image(512, 300, 'logo');

        this.title = this.add.text(512, 460, 'Main Menu', {
            fontFamily: 'Arial Black', fontSize: 38, color: '#ffffff',
            stroke: '#000000', strokeThickness: 8,
            align: 'center'
        }).setOrigin(0.5);

        this.input.once('pointerdown', () => {

            this.scene.start('Game');

        });
    }
}
```

### JavaScript Scene

```javascript
import { Scene } from 'phaser';

export class MainMenu extends Scene
{
    constructor ()
    {
        super('MainMenu');
    }

    create ()
    {
        this.add.image(512, 384, 'background');

        this.add.image(512, 300, 'logo');

        this.add.text(512, 460, 'Main Menu', {
            fontFamily: 'Arial Black', fontSize: 38, color: '#ffffff',
            stroke: '#000000', strokeThickness: 8,
            align: 'center'
        }).setOrigin(0.5);

        this.input.once('pointerdown', () => {

            this.scene.start('Game');

        });
    }
}
```

### Game Config (TypeScript)

```typescript
import { Boot } from './scenes/Boot';
import { Game as MainGame } from './scenes/Game';
import { GameOver } from './scenes/GameOver';
import { MainMenu } from './scenes/MainMenu';
import { AUTO, Game } from 'phaser';
import { Preloader } from './scenes/Preloader';

const config: Phaser.Types.Core.GameConfig = {
    type: AUTO,
    width: 1024,
    height: 768,
    parent: 'game-container',
    backgroundColor: '#028af8',
    scene: [
        Boot,
        Preloader,
        MainMenu,
        MainGame,
        GameOver
    ]
};

const StartGame = (parent: string) => {

    return new Game({ ...config, parent });

}

export default StartGame;
```
