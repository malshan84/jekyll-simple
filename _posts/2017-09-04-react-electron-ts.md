---
layout: post
title:  "Typescript 를 사용한 Electron+React 개발 환경 구축하기"
---

이번 포스팅은 https://gist.github.com/matthewjberger/6f42452cb1a2253667942d333ff53404#prerequisites 를 참고하여 작성하였습니다.

# 준비물
* [Create-React-App](https://github.com/facebookincubator/create-react-app)
* [Yarn](https://yarnpkg.com/en/)


# 설치하기 #
React app과 Electron을 다운 로드한다.

```bash
create-react-app electron-react-ts-app --scripts-version=react-scripts-ts
cd electron-react-ts-app
yarn add electron --dev
yarn add electron-builder --dev
yarn global add foreman # for process management
yarn install
```

# 소스 추가 #
`src`폴더에 `electron-starter.ts` 파일을 추가하고 아래와 같이 소스를 작성한다.
~~~typescript
import { app, BrowserWindow  } from 'electron';
import * as path from 'path';
import * as url from 'url';

let mainWindow: Electron.BrowserWindow;

function createWindow () {
    mainWindow = new BrowserWindow({width: 800, height: 600});

    const startUrl = process.env.ELECTRON_START_URL || url.format({
      pathname: path.join(__dirname, '/../build/dist/index.html'),
      protocol: 'file:',
      slashes: true
    });
    mainWindow.loadURL(startUrl);
}

app.on('ready', createWindow);

app.on('activate', function () {
  if (mainWindow === null) {
    createWindow();
  }
});
~~~
`src`폴더에 `electron-wait-react.ts`파일을 추가하고 아래와 같이 소스를 작성한다.
```typescript
import * as net from 'net';

const port: number = process.env.PORT ? (Number(process.env.PORT) - 100) : 3000;

process.env.ELECTRON_START_URL = `http://localhost:${port}`;

const client = new net.Socket();

let startedElectron = false;
const tryConnection = () => client.connect(`${port}`, () => {
        client.end();
        if (!startedElectron) {
            startedElectron = true;
            const exec = require('child_process').exec;
            exec('npm run electron');
        }
    }
);

tryConnection();

client.on('error', (error) => {
    setTimeout(tryConnection, 1000);
});
```
# tsconfig.json 수정 #
`module`의 설정을  `commonjs`로 수정한다.  
`exclude` 에서 사용하지 않는 것들 을 제거한다.
```
{
  "compilerOptions": {
    "outDir": "build/dist",
    "module": "commonjs",
    "target": "es5",
    "lib": ["es6", "dom"],
    "sourceMap": true,
    "allowJs": true,
    "jsx": "react",
    "moduleResolution": "node",
    "rootDir": "src",
    "forceConsistentCasingInFileNames": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "suppressImplicitAnyIndexErrors": true,
    "noUnusedLocals": true
  },
  "exclude": [
    "node_modules",
    "build"
  ]
}
```
typescript컴파일 명령어를 사용하여 `build/dist`폴더에 `.js`파일들이 잘 생성 되는지 확인한다.
```bash
tsc
```


# package.json 수정 #
`main`, `build`옵션은 새로 추가하고 `script`는 수정한다.
```
"main" : "build/dist/main.js",
  "homepage": "./",
  "scripts": {
    "start": "nf start -p 3000",
    "build": "react-scripts-ts build",
    "test": "react-scripts-ts test --env=jsdom",
    "eject": "react-scripts-ts eject",
    "electron": "electron .",
    "electron-build": "tsc",
    "electron-start": "node build/dist/electron-wait-react",
    "react-start": "react-scripts-ts start",
    "pack": "build --dir",
    "dist": "npm run build && build",
    "postinstall": "install-app-deps"
  },
  "build": {
    "appId": "com.electron.electron-with-create-react-app",
    "win": {
        "iconUrl": "https://cdn2.iconfinder.com/data/icons/designer-skills/128/react-256.png"
    },
    "directories": {
        "buildResources": "public"
    }
  }
```

# procfile 생성 #
루트 폴더에 `procfile`파일을 생성하고 아래와 같이 작성한다.
```
react: npm run react-start
electron: npm run electron-start
```

# 실행 #
```bash
yarn start
```