---
title: 为了了解原理，我做了一个简易的 Virtual DOM
date: 2021-04-15 14:40:19
---

## 前言

### 什么是 virtual dom？

> Virtual DOM 是一种编程概念。在这个概念里， UI 以一种理想化的，或者说“虚拟的”表现形式被保存于内存中，并通过如 ReactDOM 等类库使之与“真实的” DOM 同步。这一过程叫做协调。

> 这种方式赋予了 React 声明式的 API：您告诉 React 希望让 UI 是什么状态，React 就确保 DOM 匹配该状态。这使您可以从属性操作、事件处理和手动 DOM 更新这些在构建应用程序时必要的操作中解放出来。

> 与其将 “Virtual DOM” 视为一种技术，不如说它是一种模式，人们提到它时经常是要表达不同的东西。在 React 的世界里，术语 “Virtual DOM” 通常与 React 元素关联在一起，因为它们都是代表了用户界面的对象。而 React 也使用一个名为 “fibers” 的内部对象来存放组件树的附加信息。上述二者也被认为是 React 中 “Virtual DOM” 实现的一部分。

知道是怎么回事和知道怎么做还是有很大区别的，所以这次就尝试实现一个简易的 virtual dom。
内容主要参考自 [🎥 Building a Simple Virtual DOM from Scratch — Jason Yu](https://www.youtube.com/watch?v=85gJMUEcnkc) ，
这位老哥直接现场 coding 实现，还是很牛逼的。

废话就不多说了,let's get into it.

## 实现一些基本的 ReactDom api

包括`createElement`,`render`,`mount`

```typescript
// createElement.ts
export const createElement = (tagName: string, options: VOptions): VElement => {
  const vElem = Object.create(null); // pure object , no prototype
  Object.assign(vElem, {
    tagName,
    attrs: options.attrs,
    children: options.children,
  });
  return vElem;
};
```

```typescript
// mount.ts
export const mount = (node: HTMLElement | Text, target: HTMLElement | Text) => {
  target.replaceWith(node);
  return node;
};
```

```typescript
// render.ts
const _render = (vNode: VElement): HTMLElement => {
  const $el = document.createElement(vNode.tagName);
  const { attrs = {}, children = [] } = vNode;

  for (const [k, v] of Object.entries(attrs)) {
    $el.setAttribute(k, v);
  }
  for (const child of children) {
    $el.appendChild(render(child));
  }
  return $el;
};
export const render = (vNode: ChildrenType): HTMLElement | Text => {
  // 这里的需要判断渲染的东西是否是一个纯string
  if (typeof vNode === "string") {
    return document.createTextNode(vNode);
  }
  return _render(vNode);
};
```

在实现了这三个方法之后，就可以尝试先 render 一下了：

```typescript
// index.ts
import { mount } from "./mount";

const app = createElement("div", {
  attrs: {
    id: "app",
    name: "father",
  },
  children: [createElement("div", { children: ["just string"] })],
});

mount(render(app), document.getElementById("app"));
```

就可以看到这个了
![img.png](/images/vDom-1.png)

这样准备工作就算基本完成了。

接下来在 mount 的 app 中来个每秒变化的`count`:

```typescript
// index.ts
const createApp = (count: number) =>
  createElement("div", {
    attrs: { name: "hello", count: String(count) },
    children: [
      createElement("div", { children: [`count:${count}`] }),
      createElement("input", {}),
    ],
  });

let count = 0;
let app = createApp(count);
const root = mount(render(app), document.getElementById("app"));

setInterval(() => {
  count++;
  const newApp = createApp(count);
  mount(render(newApp));
  app = newApp;
}, 1000);
```

这时候 count 每秒都会更新，但是现在的问题在于每次都是直接更新整个 app， 导致创建的`input`明明没有发生任何变化，却依然跟着在刷新，导致无法连续输入。

## 实现`diff`

这时候就需要来一个`diff`了，先假定一下 api:

```typescript
// index.ts
setInterval(() => {
  count++;
  const newApp = createApp(count);
  const patch = diff(app, newApp);
  patch(root);
  app = newApp;
}, 1000);
```

先`diff`一下新老树的区别，然后返回一个`patch`来更新 dom。 首先要判断新节点是否存在，如果不存在就直接 remove 掉。 然后是新和旧至少有一个是`string`，这有三种情况： 如果二者之中只有一个是`string`，那节点肯定变了，直接渲染新的；如果都是`string`，再比较二者是否相同。

最后，如果 tagName 相同，就需要对比 attrs 和 children，然后再 patch 更新一下 写字也没啥用，直接上完整代码：

```typescript
// diff.ts

import { mount } from "./mount";
import { render } from "./render";

export const diff = (
  oldVNode: VElement | string,
  newVNode?: VElement | string
) => {
  if (!newVNode) {
    return (node: TElement) => {
      node.remove();
    };
  }
  if (typeof oldVNode === "string" || typeof newVNode === "string") {
    if (oldVNode !== newVNode) {
      return (node: TElement) => {
        mount(render(newVNode), node);
      };
    }
  } else if (oldVNode.tagName !== newVNode.tagName) {
    return (node: HTMLElement) => {
      mount(render(newVNode), node);
    };
  } else {
    const patchAttrs = diffAttrs(oldVNode.attrs, newVNode.attrs);
    const patchChildren = diffChildren(oldVNode.children, newVNode.children);
    return (node) => {
      patchAttrs(node);
      patchChildren(node);
    };
  }
};
```

关于`diffAttrs`我觉得没啥好说的，生成一个 patch list 然后更新就完事了。

而`diffChildren`，可以先考虑节点数相同时，那么就直接在 patch list 里面把各项`diff`即可， 如果`oldChildren.length > newChildren.length`也是一样的， 因为我们实现的`diff`接受的新 tree 可以为空

如果`oldChildren.length < newChildren.length`的话，也可以先对比到`oldChildren.length`，然后再把新树 添加的内容加上，所以前面就可以写成：

```typescript
const childPatches = [];
const additionalPatches = [];
for (let i = 0; i < oldChildren.length; i++) {
  childPatches.push(diff(oldChildren[i], newChildren[i]));
}
newChildren.slice(oldChildren.length).forEach((additionalVChild) => {
  additionalPatches.push(($node) => {
    $node.appendChild(render(additionalVChild));
  });
});
```

还需要注意的是返回的函数接受的 node 是父节点，如果在`additionalPatches`中，因为是直接`appendChild`，
而`childPatches`中的`diff`改变的应该是`parent.childNodes`。

然后在 for-loop 中需要注意删除子节点，可能会导致 i 超过`parent.childNodes`的长度，所以就可以从最后一个 childNode 开始处理，在刪除时也不会因为`childNode`被刪除，导致意外的 bug。
所以最后返回的结果为：

```typescript
return (parent: TElement) => {
  for (let i = childPatches.length - 1; i >= 0; i--) {
    const patch = childPatches[i];
    patch(parent.childNodes[i]);
  }
  for (const patch of additionalPatches) {
    patch(parent);
  }
};
```

最终的代码：

```typescript
const diffAttrs = (oldAttrs: Attrs = {}, newAttrs: Attrs = {}) => {
  const patches = [];
  for (const [k, v] of Object.entries(newAttrs)) {
    patches.push((node: HTMLElement) => {
      node.setAttribute(k, v);
    });
  }
  for (const [k] of Object.entries(oldAttrs)) {
    if (!(k in newAttrs)) {
      patches.push((node: HTMLElement) => {
        node.removeAttribute(k);
      });
    }
  }
  return (node) => {
    for (const patch of patches) {
      patch(node);
    }
  };
};
const diffChildren = (
  oldChildren: VChild[] = [],
  newChildren: VChild[] = []
) => {
  const childPatches = [];
  const additionalPatches = [];
  for (let i = 0; i < oldChildren.length; i++) {
    childPatches.push(diff(oldChildren[i], newChildren[i]));
  }
  newChildren.slice(oldChildren.length).forEach((additionalVChild) => {
    additionalPatches.push(($node) => {
      $node.appendChild(render(additionalVChild));
    });
  });

  return (parent: TElement) => {
    for (let i = childPatches.length - 1; i >= 0; i--) {
      const patch = childPatches[i];
      patch(parent.childNodes[i]);
    }
    for (const patch of additionalPatches) {
      patch(parent);
    }
  };
};
```

所以这样就算是完成了，[完整代码在这里](https://github.com/debugtheworldbot/virtual-dom) ,
虽然是非常非常简略的 virtual dom，还有各种更新 view 的 case 没有 cover 到，但是整个流程下来还是学到了不少的，
