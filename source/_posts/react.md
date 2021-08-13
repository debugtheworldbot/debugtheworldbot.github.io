---
title: build your own react from scratch
date: 2021-08-13 10:53:57
tags: react
---

## Step 0:简介

本文章从 `createElement` 函数讲起，然后一步一步完成了一个简单的 Fiber 、 Reconclie 、 commitRoot 等 react 的核心部分，最终实现一个 react。

如果你比较熟悉 react，dom，jsx 是如何工作的，可以跳过本节。
我们从最基本的`vanilla JavaScript`开始。

这是一段 jsx 语法：

```jsx
const element = <h1 title="foo">Hello</h1>;
```

jsx 会被类似于 Babel 之类的工具转换成为 js，上面的 jsx 会被转化为：

```js
const element = React.createElement("h1", { title: "foo" }, "Hello");
```

而一个 virtual element 应该是一个带有 type 和 props 的对象:

```js
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
};
```

其中的 type 即是`document.createElement`传入的 tagName，还有一种情况是它也可能是一个`function`，这个将在后面处理。
props 是另一个含有各种 key 和 value 的对象以及一个特殊的 property：children,children 可以是 string 也可以是一个 element 数组。

最后，还需要一个`render`函数，把 element 挂载到真实的 dom 上，render 大概是做这些事情,创建 node，加入 props，然后挂载 children:

```js
const container = document.getElementById("root")
​
const node = document.createElement(element.type)
node["title"] = element.props.title
​
const text = document.createTextNode("")
text["nodeValue"] = element.props.children
​
node.appendChild(text)
container.appendChild(node)

```

## Step 1:createElement

这里需要注意如果 children 是个 string 的话，应该是创建一个`TEXT_ELEMENT`

```js
// createElement.ts

const createElement = (
  type: string,
  props: Partial<HTMLElement>,
  ...children: VElement[]
): VElement => {
  return {
    type,
    props: {
      ...props,
      children: children
        // 这里使用flat的原因是如果jsx里面包含map返回的一个element数组，就需要把children给拍平
        .flat()
        .map((c) => (typeof c === "object" ? c : createTextElement(c))),
    },
  };
};

const createTextElement = (text: string): VElement => {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
};
```
