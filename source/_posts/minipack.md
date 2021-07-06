---
title: minipack
date: 2021-07-06 10:23:40
tags: javascript
---

# intro

什么是 Modules？

> 在模块化编程中，开发者将程序分解为功能离散的 chunk，并称之为 **Modules**。

> 每个 Module 都拥有小于完整程序的体积，使得验证、调试及测试变得轻而易举。精心编写的 Module 提供了可靠的抽象和封装界限，使得应用程序中每个模块都具备了条理清晰的设计和明确的目的。

`Module bundlers` 应该需要知道入口文件的信息，也就是我们实例的主文件，它将引导启动整个实例。

所以我们的 bundler 应该从入口文件开始，分析它的依赖，以及依赖的依赖...一直递归到最后一层。而这整个过程最终就会形成一个`依赖图谱`。

Let's go

note: 这是一个非常简单的栗子。像检查循环引用，对于 module 只 parse 一次等等的操作都是没有的。所以 100 多行代码就能搞定。

# overview

```js
// entry.js
import world from "./world.js";

console.log(world);
```

```js
// world.js
import { hello } from "./hello.js";

export default `${hello} from world`;
```

```js
// hello.js
export const hello = "hello";
```

所以它们的引用顺序就是 entry->world->hello ，然后就可以从入口文件开始解析，写代码了

# create asset

create asset 应该要做以下几件事情:

- 读入口文件，转换成 ast
- 获取当前文件 import 的文件
- 转换成 commonjs 的代码，使其能在浏览器运行
- 给每个文件一个 id

因为 EcmaScript modules 是静态的，它不会导出一个变量，或者有条件的引入某些 module，所以在这里的每一个 import 语句都可以存在统一的 dep 数组里面

```js
const fs = require("fs");
const path = require("path");
const { parse } = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const { transformFromAst } = require("@babel/core");

let ID = 0

const createAsset = (path) => {
  const code = fs.readFileSync(path, "utf-8");
  const ast = parse(code, {
    sourceType: "module",
  });
  const deps = []; // 文件的相对依赖
  traverse(ast, {
    ImportDeclaration: ({ node }) => {
      deps.push(node.source.value); // 这个值为 ‘./world.js’ | './hello.js'
    },
  });
  const { code: transformedCode } = transformFromAst(ast, null, {
    presets: ["@babel/preset-env"], // need add this dependency
  });
  return {
      id:ID++
      code:transformedCode,
      deps,
      path
  }
};
```

这时候 `createAsset(./src/entry.js)`返回值就是：

```js
{
  id: 0,
  path: './src/entry.js',
  code: '"use strict";\n' +
    'var _world = _interopRequireDefault(require("./world.js"));\n' +
    'function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n' +
    'console.log(_world["default"]);',
  deps: [ './world.js' ]
}
```

也可以试试`hello.js`和`world.js`:

```js
{
  id: 1,
  path: './src/world.js',
  code: '"use strict";\n' +
    'Object.defineProperty(exports, "__esModule", {\n' +
    '  value: true\n' +
    '});\n' +
    'exports["default"] = void 0;\n' +
    'var _hello = require("./hello.js");\n' +
    'var _default = "".concat(_hello.hello, " from world");\n' +
    'exports["default"] = _default;',
  deps: [ './hello.js' ]
}
{
  id: 2,
  path: './src/hello.js',
  code: '"use strict";\n' +
    'Object.defineProperty(exports, "__esModule", {\n' +
    '  value: true\n' +
    '});\n' +
    'exports.hello = void 0;\n' +
    'var hello = "hello";\n' +
    'exports.hello = hello;',
  deps: []
}
```
