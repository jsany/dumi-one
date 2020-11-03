# lean

> dumi babel exapple （base father2）

## babel 方式踩的坑

> babel 方式是文件到文件的编译，不会做额外的处理

- 不要在组件中使用 cssModules，否则，构建产物也会保持 cssModules 的引用方式（形如：`import Styles from 'style/index.css`）
- 需要生成 d.ts 文件，需要引入静态资源，看 ~~[PR](https://github.com/umijs/father/pull/220)~~，[临时方案](https://github.com/umijs/father/issues/227)
- 如果需要 umd 产物，不要像 antd 那样 css 和 js 分离开发，因为 umd 使用的是 rollup 打包，分离开发下 css 没有依赖关系，不会被打包

## Introduction

- base antd
- babel
- build esm、cjs、umd
- support import assets: css、image、...
- supprot d.ts
- support unitTest(@testing-library/react)

### 方案

### support import assets && supprot d.ts

为了支持 babel 方式下，引入静态资源，并生成 d.ts 声明文件，~~提了 [PR](https://github.com/umijs/father/pull/220)，但 father2 目前官方没时间处理，我这里进行了 hack 处理。~~

临时方案：<https://github.com/umijs/father/issues/227>，以下不用看了

---

~~增加 npm 命令 `"hack": "node scripts/hack-depend.js"`，对 node_modules 下的依赖进行 hack 处理：~~

- 在 `typings.d.ts` 文件中声明非 `js/ts` 模块：

```ts
declare module '*.css';
declare module '*.less';
declare module '*.png';
declare module '*.jpg';
declare module '*.gif';
```

- `package.json` 添加脚本命令：

```json
"scripts" : {

  "postinstall": "node scripts/hack-depend.js"
}
```

- 写 `scripts/hack-depend.js` 脚本：

```js
/**
 * 重写依赖模块（读取文件查找替换）
 */

const fs = require('fs');
const path = require('path');
const sep = path.posix.sep;

const hacks = [
  {
    // https://github.com/umijs/father/pull/220 ,官方没时间处理 PR，这里进行 hack 处理
    name: 'father-build',
    path: '../node_modules/father-build/lib/babel.js',
    hack: data => {
      // console.log(data)
      return data
        .replace(
          `
    function getTsconfigCompilerOptions(path) {
      const config = parseTsconfig(path);
      return config ? config.compilerOptions : undefined;
    }

    function getTSConfig() {
      const tsconfigPath = (0, _path.join)(cwd, 'tsconfig.json');
      const templateTsconfigPath = (0, _path.join)(__dirname, '../template/tsconfig.json');

      if ((0, _fs.existsSync)(tsconfigPath)) {
        return getTsconfigCompilerOptions(tsconfigPath) || {};
      }

      if (rootPath && (0, _fs.existsSync)((0, _path.join)(rootPath, 'tsconfig.json'))) {
        return getTsconfigCompilerOptions((0, _path.join)(rootPath, 'tsconfig.json')) || {};
      }

      return getTsconfigCompilerOptions(templateTsconfigPath) || {};
    }`,
          `
    function getTsconfigCompilerOptions(path) {
      const config = parseTsconfig(path);
      return config ? config.compilerOptions : undefined;
    }

    function getTsconfigInclude(path) {
      const config = parseTsconfig(path);
      const includesPath = config ? config.include || [] : [];
      return includesPath;
    }

    function getTSConfig() {
      const tsconfigPath = (0, _path.join)(cwd, 'tsconfig.json');
      const templateTsconfigPath = (0, _path.join)(__dirname, '../template/tsconfig.json');

      if ((0, _fs.existsSync)(tsconfigPath)) {
        return getTsconfigCompilerOptions(tsconfigPath) || {};
      }

      if (rootPath && (0, _fs.existsSync)((0, _path.join)(rootPath, 'tsconfig.json'))) {
        return getTsconfigCompilerOptions((0, _path.join)(rootPath, 'tsconfig.json')) || {};
      }

      return getTsconfigCompilerOptions(templateTsconfigPath) || {};
    }

    function getTSMatch() {
      const tsconfigPath = join(cwd, 'tsconfig.json');
      const templateTsconfigPath = join(__dirname, '../template/tsconfig.json');
      if (existsSync(tsconfigPath)) {
        return getTsconfigInclude(tsconfigPath) || [];
      }
      if (rootPath && existsSync(join(rootPath, 'tsconfig.json'))) {
        return getTsconfigInclude(join(rootPath, 'tsconfig.json')) || [];
      }
      return getTsconfigInclude(templateTsconfigPath) || [];
    }`,
        )
        .replace(
          `&& !path.endsWith('.d.ts');`,
          `&& (path.endsWith('typings.d.ts') || path.endsWith('index.d.ts') || !path.endsWith('.d.ts'));`,
        )
        .replace(
          `[(0, _path.join)(srcPath, '**/*'),`,
          `[(0, _path.join)(srcPath, '../typings.d.ts'),(0, _path.join)(srcPath, '../index.d.ts'),(0, _path.join)(srcPath, '../typings/index.d.ts'),(0, _path.join)(srcPath, '**/*'),`,
        );
    },
  },
];

const run = () => {
  for (const item of hacks) {
    const finalPath = path.resolve(__dirname, item.path.replace(/\//g, sep));
    fs.readFile(finalPath, 'utf8', (err, data) => {
      if (err) {
        throw err;
      }
      // console.log('=====data=====')
      // console.log(data.indexOf(`&& !path.endsWith('.d.ts');`))
      // console.log(typeof data)
      // console.log(String(data))
      // console.log('=====data=====')

      const fixed = item.hack(data);
      fixed &&
        fs.writeFile(finalPath, fixed, err => {
          if (err) {
            throw err;
          }
          console.log('hack success');
        });
    });
  }
};

run();
```

---

## 发包后，在 umi 项目中使用 lean 组件库

### 在 tsx 中使用

```tsx
import React from 'react';
import { Foo, Button } from 'lean';

export default () => {
  return (
    <div>
      <Foo title="demo" />
      <Button btnType="primary">primary</Button>
    </div>
  );
};
```

## Getting Started

Install dependencies,

```bash
$ npm i
```

~~Hack dependencies,~~

```bash
$ npm run hack
```

Start the dev server,

```bash
$ npm start
```

Build documentation,

```bash
$ npm run docs:build
```

Build library via `father-build`,

```bash
$ npm run build
```

Test Components,

```bash
# test all components
$ npm run test
```

```bash
# test all components with coverage
$ npm run test:coverage
```

```bash
# test one component Foo
$ npx umi-test src/Foo/index.test.tsx
```
