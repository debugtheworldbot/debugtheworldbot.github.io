---
title: minipack - 一个极简webpack
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

这个例子的引用顺序就是 entry->world->hello ，我们从入口文件开始解析

# create asset

create asset 应该要做以下几件事情:

- 读入口文件，转换成 ast
- 获取当前文件 import 的文件
- 给每个文件一个 id
- 转换成 commonjs 的代码，使其能在浏览器运行

因为 EcmaScript modules 是静态的，它无法导出一个变量，或者有条件的引入某些 module，所以在这里的每一个 import 语句都可以存在统一的 dep 数组里面

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

也可以试试`hello.js`和`world.js`，它们是这样的：

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

# create graph

在获取到 createAsset 的返回值之后，就可以根据返回值生成 dependancy graph，这个函数将依次做以下几件事情：

- 获取 mainAssets，并将所有的 asset 保存在一个数组(`asstes`)中
- 遍历这个`assets`,同时去读取它的 deps，如果 dep 不为空，就把读取的结果 push 到`assets`中
- 同时生成一个 mapping 来储存 asset 和它的 children 的关系，key 为 deps 里面的路径，value 为当前 asset 的 child 的 id
- 返回`assets`图谱

```js
const createGraph = (entry) => {
  const mainAssets = createAsset(entry);
  const assets = [mainAssets]; // 初始化assets
  for (const asset of assets) {
    const { deps, path } = asset;
    const dir = path.dirname(path); // 获取绝对路径
    asset.mapping = {};
    for (const dep of deps) {
      const child = createAsset(path.join(dir, dep));
      asset.mapping[dep] = child.id;
      assets.push(child); // 因为是在这里push，所以assets的循环可以一直继续，直到deps为空数组结束
    }
  }
  return assets;
};
```

这时候返回的`assets`为：

```js
[
  {
    id: 0,
    path: "./src/entry.js",
    code:
      '"use strict";\n' +
      'var _world = _interopRequireDefault(require("./world.js"));\n' +
      'function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }\n' +
      'console.log(_world["default"]);',
    deps: ["./world.js"],
    mapping: { "./world.js": 1 },
  },
  {
    id: 1,
    path: "src/world.js",
    code:
      '"use strict";\n' +
      'Object.defineProperty(exports, "__esModule", {\n' +
      "  value: true\n" +
      "});\n" +
      'exports["default"] = void 0;\n' +
      'var _hello = require("./hello.js");\n' +
      'var _default = "".concat(_hello.hello, " from world");\n' +
      'exports["default"] = _default;',
    deps: ["./hello.js"],
    mapping: { "./hello.js": 2 },
  },
  {
    id: 2,
    path: "src/hello.js",
    code:
      '"use strict";\n' +
      'Object.defineProperty(exports, "__esModule", {\n' +
      "  value: true\n" +
      "});\n" +
      "exports.hello = void 0;\n" +
      'var hello = "hello";\n' +
      "exports.hello = hello;",
    deps: [],
    mapping: {},
  },
];
```

看起来有点样子了,接下来就可以把代码写到文件里面啦~~

# bundle

根据生成的 graph，就可以把一片片的 code 打包起来了，它返回的是一个立即执行函数，也就是可以直接在浏览器里面运行。
也就是这样子：`(;function(modules){})(modules)`

它需要接受一个参数`modules`：一个带有每一个 graph 信息的对象，也就是这样:

```js
{
    id:[fn(require,module,exports){code},mapping]
}
```

id 数组的第一项储存 code，它们都被一个 function 包裹，这样就不会有全局变量污染的问题，这个 function 用的是 commonJS 的形式，所以它需要三个参数 `require module exports`，需要我们构造一下加进去

id 数组的第二项就是之前构造的 mapping，大概长这样：`mapping: { "./relative/path": 2 }`，也可以往上翻看下生成的 graph

require 就是请求对应的 code，根据上面的 graph，我们可以让 `require(id)` 直接获取到对应的 code，但是在生成的代码中，require 应该接受的是一个**相对路径**，这时候我们之前的 mapping 就终于有用了，它刚好就是相对路径和 id 的映射。

最后，在 commonJS 中，当一个 module 被引入的时候，它会改变自己的 exports 对象，最终被`require()`返回

所以 require 就可以这样实现：

```js
function require(id) {
  const [fn, mapping] = modules[id];
  function localRequire(path) {
    return require(mapping[path]);
  }
  const module = { exports: {} };
  fn(localRequire, module, module.exports);
  return module.exports;
}
```

最终的 bundle：

```js
const buldle = (graph) => {
  let modules = "";
  grpah.forEach((g) => {
    modules += `${g.id}:[function(require,module,exports){
          ${g.code}
      },${JSON.stringfy(g.mapping)}],`;
  });

  return `
    ;(function(modules){
        function require (id){
            const [fn,mapping] = modules[id]
            function localRequire(path){
                return require(mapping[path])
            }
            const module = {exports:{}}
            fn(localRequire,module,module.exports)
            return module.exports
        }
        require(0)
    }({${modules}})) // 这里传的是一个对象
  `;
};
```

然后在最后再把 bundle 的代码写入 dist：

```js
const graph = createGraph("./src/entry.js");
const result = bundle(graph);
fs.writeFileSync("./dist/bundle.js", result, { encoding: "utf-8" });
```

把生成的 js 代码放到浏览器里执行：
![result](/images/minipack.png)

AWESOME!

if you want to know more , [this is the source code](https://github.com/debugtheworldbot/miniWebpack)
