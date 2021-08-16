---
title: 实现 react
date: 2021-08-13 10:53:57
tags: react
---

## Step 0: 简介

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

## Step 1: createElement

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

## Step 2: render

目前只考虑一种最基本的情况，即添加元素，后面会加入对 element 的更新和删除
添加的过程很简单，就是把 property 依次添加上，然后*递归*地 render children。

```jsx
const render = (element: VElement, container: HTMLElement) => {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)
​
  // 排除 特殊属性 "children"
  const isProperty = key => key !== "children"

  // 将元素属性 一一 写入 dom 节点上
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name]
    })

​  // 遍历递归 将 子元素 一个一个 都 附到 真实的 dom 节点上
  element.props.children.forEach(child =>
    render(child, dom)
  )
​
  // 最后挂载到 指定的 dom 节点容器上
  container.appendChild(dom)
}

```

## Step 3: Concurrent mode

对于上一步实现的 render，实际上是有点问题的，就是在于上面的那个*递归*，只要开始了 render，那么直到它完成，整个 render 才会结束，如果 element tree 比较庞大，render 过程中可能会阻塞浏览器进程很长时间，就会影响浏览器更高优先级的事务处理（比如用户的输入和 ui 交互等等）

所以我们需要把这个大的任务切割为多个小的工作单元，这样的话，如果浏览器有更高优先级的事务处理，我们就可以中断 react 元素的渲染，这我们引入一个概念，称它为 [并发模式](https://zh-hans.reactjs.org/docs/concurrent-mode-intro.html#:~:text=%E8%AE%A9%E6%88%91%E4%BB%AC%E5%9B%9E%E9%A1%BE%E4%B8%80%E4%B8%8B%E4%B8%8A%E9%9D%A2%E7%9A%84%E4%B8%A4%E4%B8%AA%E4%BE%8B%E5%AD%90%E7%84%B6%E5%90%8E%E7%9C%8B%E4%B8%80%E4%B8%8B%20Concurrent%20%E6%A8%A1%E5%BC%8F%E6%98%AF%E5%A6%82%E4%BD%95%E5%B0%86%E5%AE%83%E4%BB%AC%E8%81%94%E5%90%88%E8%B5%B7%E6%9D%A5%E7%9A%84%E3%80%82%E5%9C%A8%20Concurrent%20%E6%A8%A1%E5%BC%8F%E4%B8%AD%EF%BC%8CReact%20%E5%8F%AF%E4%BB%A5%20%E5%90%8C%E6%97%B6,%E8%8A%82%E7%82%B9%E5%92%8C%E8%BF%90%E8%A1%8C%E7%BB%84%E4%BB%B6%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%A0%81) ：

> 在 Concurrent 模式中，React 可以 同时 更新多个状态 —— 就像分支可以让不同的团队成员独立地工作一样：
> 对于 CPU-bound 的更新 (例如创建新的 DOM 节点和运行组件中的代码)，并发意味着一个更急迫的更新可以“中断”已经开始的渲染。
> 对于 IO-bound 的更新 (例如从网络加载代码或数据)，并发意味着 React 甚至可以在全部数据到达之前就在内存中开始渲染，然后跳过令人不愉快的空白加载状态。

![concurrent mode](/images/react-1.png)

我们可以用[requestIdleCallback](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)来实现此功能。React 目前不再使用`requestIdleCallback`，而是[scheduler package](https://pomb.us/build-your-own-react/)，不过从概念上来说这两个是一样的。

```js

let nextUnitOfWork = null
​
function workLoop(deadline) {
 // 是否要暂停
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    // 执行 一个工作单元 并返回下一个工作单元
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    // 判断空闲时间是否足够
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
​
requestIdleCallback(workLoop)
​
function performUnitOfWork(nextUnitOfWork) {
  // TODO
}

```

## Step 4: Fibers

假设 render 的 tree 是这样：

```jsx
render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
);
```

那么最开始，我们会将 root fiber 设置为上面的`nextUnitOfWork`，剩下的事情就会在`performUnitOfWork`中完成，主要是做三件事：

1. 把 element 添加到 DOM 中
2. 为 element 的每一个 child 创建 fiber
3. 指定下一个 nextUnitOfWork

为了让每一个 fiber 能够方便的找到下一个 `nextUnitOfWork`，它本身应该包含它的第一个 child，下一个相邻的元素以及它的父亲，数据结构为：

```ts
interface VElement {
  type?: string | "TEXT_ELEMENT" | Function;
  props: HTMLProps;

  parent?: VElement | null;
  child?: VElement | null;
  sibling?: VElement | null;
}
interface HTMLProps extends Partial<Omit<HTMLElement | Text, "children">> {
  nodeValue?: string | null;
  children: VElement[];
}
```

而遍历的时候采用的是深度优先遍历，即先从 root **递** 到主 tree 的最后一个 first child，然后检查它是否有 sibling，当 siblings 遍历结束时再去遍历 parent 的 siblings...依次再 **归** 回到 root

```js
const createDom = (fiber) => {
    const dom = fiber.type === 'TEXT_ELEMENT' ?
        document.createTextNode('') : document.createElement(fiber.type as string)
    return dom
}
const performUnitOfWork = (fiber)=> {
  // 创建一个 dom 元素，挂载到 fiber 的 dom 属性
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​  // 添加 dom 到父元素上
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​
  const elements = fiber.props.children
  let index = 0
  // 保存上一个 sibling fiber 结构
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }
​    // 第一个子元素 作为 child，其余的子元素都作为 sibling
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }
​  // 如果有 child fiber ，则返回 child
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber

  while (nextFiber) {
    // 如果有 sibling fiber，则返回 sibling
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    // 否则返回它的 parent fiber
    nextFiber = nextFiber.parent
  }
}
```

## Step 5: Render and Commit

目前虽然是实现了可中断的 render 了，但是还有个问题就在它们是每次直接把 element 添加到 DOM 上，如果在这个过程中浏览器暂时中断了渲染的过程，那么就会呈现出不完整的 UI，这是非常不好的，在 react 官方文档中对 `error boundary`的处理不完整的 UI 有这样的[评论](https://zh-hans.reactjs.org/docs/error-boundaries.html#:~:text=%E6%88%91%E4%BB%AC%E5%AF%B9%E8%BF%99%E4%B8%80%E5%86%B3%E5%AE%9A%E6%9C%89%E8%BF%87%E4%B8%80%E4%BA%9B%E4%BA%89%E8%AE%BA%EF%BC%8C%E4%BD%86%E6%A0%B9%E6%8D%AE%E6%88%91%E4%BB%AC%E7%9A%84%E7%BB%8F%E9%AA%8C%EF%BC%8C%E6%8A%8A%E4%B8%80%E4%B8%AA%E9%94%99%E8%AF%AF%E7%9A%84%20UI%20%E7%95%99%E5%9C%A8%E9%82%A3%E6%AF%94%E5%AE%8C%E5%85%A8%E7%A7%BB%E9%99%A4%E5%AE%83%E8%A6%81%E6%9B%B4%E7%B3%9F%E7%B3%95%E3%80%82%E4%BE%8B%E5%A6%82%EF%BC%8C%E5%9C%A8%E7%B1%BB%E4%BC%BC%20Messenger%20%E7%9A%84%E4%BA%A7%E5%93%81%E4%B8%AD%EF%BC%8C%E6%8A%8A%E4%B8%80%E4%B8%AA%E5%BC%82%E5%B8%B8%E7%9A%84%20UI%20%E5%B1%95%E7%A4%BA%E7%BB%99%E7%94%A8%E6%88%B7%E5%8F%AF%E8%83%BD%E4%BC%9A%E5%AF%BC%E8%87%B4%E7%94%A8%E6%88%B7%E5%B0%86%E4%BF%A1%E6%81%AF%E9%94%99%E5%8F%91%E7%BB%99%E5%88%AB%E4%BA%BA%E3%80%82%E5%90%8C%E6%A0%B7%EF%BC%8C%E5%AF%B9%E4%BA%8E%E6%94%AF%E4%BB%98%E7%B1%BB%E5%BA%94%E7%94%A8%E8%80%8C%E8%A8%80%EF%BC%8C%E6%98%BE%E7%A4%BA%E9%94%99%E8%AF%AF%E7%9A%84%E9%87%91%E9%A2%9D%E4%B9%9F%E6%AF%94%E4%B8%8D%E5%91%88%E7%8E%B0%E4%BB%BB%E4%BD%95%E5%86%85%E5%AE%B9%E6%9B%B4%E7%B3%9F%E7%B3%95%E3%80%82)：

> 我们对这一决定有过一些争论，但根据我们的经验，把一个错误的 UI 留在那比完全移除它要更糟糕。例如，在类似 Messenger 的产品中，把一个异常的 UI 展示给用户可能会导致用户将信息错发给别人。同样，对于支付类应用而言，显示错误的金额也比不呈现任何内容更糟糕。

所以我们需要做出一些些改动，当**所有**工作单元执行完后，再一并将所有的 dom 的添加

```ts
export const workLoop = (deadline: TimeRemaining) => {
  let shouldYeild = false;
  while (!!nextUnitOfWork && !shouldYeild) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYeild = deadline.timeRemaining() < 1;
  }
  // 等到nextUnitOfWork做完之后再一并提交到 DOM
  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }
  window.requestIdleCallback(workLoop);
};

const commitRoot = () => {
  commitWork(wipRoot?.child!);
  wipRoot = null;
};
const commitWork = (fiber?: Fiber) => {
  if (!fiber) return;
  let domParentFiber = fiber.parent;
  while (!domParentFiber?.dom) {
    domParentFiber = domParentFiber?.parent;
  }
  const domParent = domParentFiber?.dom;
  domParent?.appendChild(fiber.dom);
  commitWork(fiber.child);
  commitWork(fiber.sibling);
};
```

## Step 6: Reconciliation

之前处理的都是向 DOM 之中添加 elements，其实还有两种操作：`update`和`delete`，所以在 commit 结束之后，需要保存当前的 fiber tree (用`currentRoot`来保存)，等到下次 render 时与 wipRoot(work in progress)的 fiber 树进行比较，同时在 wipRoot 中增加一个 alternate 属性来连接旧的 fiber 树。

同时在每次 commit 的时候执行加入 deletions 列表，然后在重新 render 时重置 deletions

```ts
const commitRoot = () => {
  deletions?.forEach((d) => commitWork(d));
  commitWork(wipRoot?.child!);
  // 保存当前的tree
  currentRoot = wipRoot;
  wipRoot = null;
};
export const render = (element: VElement, container: HTMLElement) => {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  };
  // 重置deletions
  deletions = [];
  nextUnitOfWork = wipRoot;
};
```

对于`performUnitOfWork`，我们把对 fiber 的遍历放到 reconcile 中

```ts
const performUnitOfWork = (fiber: VElement) => {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }
  const elements = fiber.props.children;
  // 在这里操作fiber
  reconcileChildren(fiber, elements);

  if (fiber.child) return fiber.child;

  let nextFiber = fiber;
  // if sibling exists,return;else return parent
  while (nextFiber) {
    if (nextFiber.sibling) return nextFiber.sibling;
    nextFiber = nextFiber.parent!;
  }
  return null;
};

const reconcileChildren = (fiber: Fiber, elements: VElement[]) => {
  if (!fiber) return;
  let index = 0;
  // 获取到旧fiber
  let oldFiber = fiber.alternate?.child;
  let preSibling: VElement | null = null;

  while (index < elements?.length || !!oldFiber) {
    const element = elements[index];

    const sameType = oldFiber && element && element.type === oldFiber.type;

    let newFiber: VElement | null = null;
    if (sameType) {
      // update
      newFiber = {
        type: oldFiber!.type,
        props: element.props,
        // 使用之前的dom，只需要更新props
        dom: oldFiber?.dom,
        parent: fiber,
        alternate: oldFiber,
        // effectTag将在commit阶段使用
        effectTag: "UPDATE",
      };
    } else if (!sameType && element) {
      // add
      newFiber = {
        type: element.type,
        props: element.props,
        // 需要创建一个新的dom
        dom: null,
        parent: fiber,
        alternate: null,
        effectTag: "PLACEMENT",
      };
    } else {
      // remove
      oldFiber!.effectTag = "DELETION";
      // 收集deletions，在commitRoot时统一移除
      deletions!.push(oldFiber!);
    }
    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      preSibling!.sibling = newFiber;
    }
    preSibling = newFiber;
    index++;
  }
};
```

然后，在 commit 阶段来处理之前加上的`effectTags`，其中的 UPDATE 即为更新当前 fiber 的 props，包括简单属性的更新(setProperty)以及以 on 开头的事件的更新(addEventListener)

```ts
const commitWork = (fiber?: Fiber) => {
  if (!fiber) return;
  let domParentFiber = fiber.parent;
  while (!domParentFiber?.dom) {
    domParentFiber = domParentFiber?.parent;
  }
  const domParent = domParentFiber?.dom;
  if (fiber.effectTag === "PLACEMENT" && !!fiber.dom) {
    // 如果是添加，就直接添加
    domParent?.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && !!fiber.dom) {
    // update fiber
    updateDom(fiber.dom, fiber.alternate?.props!, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    let child = fiber;
    while (!child?.dom) {
      child = fiber.child!;
    }
    domParent?.removeChild(child.dom!);
  }
  // 继续commit child 和 sibling
  commitWork(fiber.child);
  commitWork(fiber.sibling);
};

const isNew = (prev: HTMLProps | {}, next: HTMLProps) => (key: string) =>
  prev[key] !== next[key];
const isGone = (prev: HTMLProps | {}, next: HTMLProps) => (key: string) =>
  !(key in next);
const isEvent = (key: string) => key.startsWith("on");
const isProperty = (key: string) => key !== "children" && !isEvent(key);

export const updateDom = (
  dom: HTMLElement | Text,
  prevProps: HTMLProps | {},
  nextProps: HTMLProps
) => {
  // Remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = "";
    });
  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = nextProps[name];
    });
  //Remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });
  // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });
};
```

## Step 7: Function Component & useState

如果我们的例子是这样：

```ts
import { createElement } from "./react/createElement";

/** @jsx createElement */
function App(props) {
  return <h1>Hi {props.name}</h1>;
}
const element = <App name="foo" />;
const container = document.getElementById("root");
render(element, container);
```

再转译之后变成 js 将会是这样:

```ts
function App(props) {
  return createElement("h1", null, "Hi ", props.name);
}
const element = createElement(App, {
  name: "foo",
});
```

所以在执行 reconcile 之前，应该检查一下 fiber 的 type 是否是 Function,如果的话 fiber 的 children 应该通过调用 `fiber.type`来获得:

```ts
let wipFiber: VElement;
let hookIndex: number = -1;
const isFucComponent = fiber.type instanceof Function;
if (isFucComponent) {
  wipFiber = fiber;
  hookIndex = 0;
  wipFiber.hooks = [];
  const children = [(fiber.type as Function)(fiber.props)];
  reconcileChildren(fiber, children);
} else {
  const elements = fiber.props.children;
  reconcileChildren(fiber, elements);
}
```

关于 useState 的实现,我们用一个数组来存 hooks，同时用一个 hookIndex 来追踪状态，每一个 hook 中保存着前一个 state 和一个更新 state 的函数列表，然后调用 setState 时指定 `nextUnitOfWork` 触发 actions 更新 state。

```ts
export const useState = <T>(initial: T): [T, Function] => {
  const oldHook = wipFiber?.alternate?.hooks![hookIndex];
  const hook: { state: T; queue: Function[] } = {
    state: oldHook ? oldHook.state : initial,
    // 存放每次更新状态的队列
    queue: [],
  };
  const actions = oldHook ? oldHook.queue : [];
  actions.forEach(
    (action) =>
      (hook.state = action instanceof Function ? action(hook.state) : action)
  );
  const setState = (action: Function) => {
    hook.queue.push(action);
    wipRoot = {
      dom: currentRoot?.dom,
      props: currentRoot?.props!,
      alternate: currentRoot,
    };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };
  wipFiber.hooks?.push(hook);
  hookIndex++;

  return [hook.state, setState];
};
```

## 小结

这个就是本文构建的 react，这里是[源码](https://github.com/debugtheworldbot/redesigned-react)和[在线 demo](https://debugtheworldbot.github.io/redesigned-react/)

为了便于理解 react 是如何工作的，这份代码用了和 react 源码里面一模一样的变量名和函数名，like:

- workLoop
- performUnitOfWork
- updateFunctionComponent

但是在有些地方和 react 本身的实现会有一些出入：

- 在 render 和 commit 阶段，我们都会遍历整个 fiber tree，而 react 会选择性地跳过一些并没有发生变化的子树
- 每一次我们构建一个新的`work in progress`tree 时，都会为每一个 fiber 创建一个新对象,而 react 会回收之前 tree 里面的一些 fiber
- 在 render 阶段触发更新时，我们会丢弃掉整个`work in progress` tree，然后从 root 重新开始，而 react 会给每个更新打一个有过期时间的 tag，并且依赖它来确定哪一个更新具有更高的优先级
- ...

总的代码量也就 200 多行，但是对于了解 react 大体上是如何工作的还是有很大帮助的。
