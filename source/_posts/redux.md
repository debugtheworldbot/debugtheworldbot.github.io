---
title: 通过context实现redux
date: 2021-04-13 14:59:58
tags: react
---

## 先尝试实现一个简单的全局 store 和 setStore

先看页面：

![1](/images/redux-1.png)

目前的实现非常简单，直接看代码：

```jsx
// App.jsx
import React from "react";
import { connect, appContext, store } from "./redux";

export const App = () => {
  console.log("app");
  return (
    <appContext.Provider value={store}>
      {" "}
      //store里面存的就是全局的数据
      <Grandfather /> // 只是单纯的展示store里面的数据
      <Father /> // 可以修改store中的数据
      <Son /> // 完全静态的组件
    </appContext.Provider>
  );
};

const Grandfather = () => {
  console.log("grandfather");
  return (
    <section>
      Grandfather
      <User />
    </section>
  );
};
const User = connect(({ state }) => {
  console.log("user");
  return <div>User:{state.user.name}</div>;
});

const Father = () => {
  console.log("father");
  return (
    <section>
      father <UserModifier />
    </section>
  );
};
const Son = () => {
  console.log("son");
  return <section>son</section>;
};

const _UserModifier = ({ dispach, state }) => {
  console.log("usermocidifer");
  const onChange = (e) => {
    dispach({ type: "updateUser", payload: { name: e.target.value } });
  };
  return (
    <div>
      <input value={state.user.name} onChange={onChange} />
    </div>
  );
};
const UserModifier = connect(_UserModifier);
```

`store`,`dispatch`和`connect`都写在 redux.jsx 中

为了避免不必要的渲染，所以在 store 里面加上了订阅模式,
订阅时需要传入更新的函数（setState）使对应的 ui 更新，在`store.setState`的时候就会执行 listener list 中的函数，
在对应组件销毁时时会清除 listener 里面的更新函数

`reducer`没啥好说的，就是更新`store.state`里面的数据

```jsx
// reudx.jsx

import React, { useState, useContext, useEffect } from "react";

export const appContext = React.createContext(null);

export const store = {
  state: {
    user: { name: "frank", age: 18 },
  },
  setState: (newState) => {
    store.state = newState;
    store.linsteners.map((fn) => fn(store.state));
  },
  linsteners: [],
  subscribe: (fn) => {
    store.linsteners.push(fn);
    return () => {
      const newList = store.linsteners.filter(
        (f) => JSON.stringify(f) !== JSON.stringify(fn)
      );
      store.linsteners = newList;
    };
  },
};
// reducer 即为更新state数据

const reducer = (state, { type, payload }) => {
  if (type === "updateUser") {
    return {
      ...state,
      user: {
        ...state.user,
        ...payload,
      },
    };
  } else {
    return state;
  }
};

export const connect = (Compenent) => {
  return (props) => {
    const { state, setState, subscribe } = useContext(appContext);
    const [, update] = useState({});
    const dispach = (action) => {
      setState(reducer(state, action));
    };
    useEffect(() => {
      const clean = subscribe(() => update({}));
      return () => {
        clean();
      };
    }, []);
    return <Compenent {...props} dispach={dispach} state={state} />;
  };
};
```

`connect`用于连接组件,返回一个被连接到 store 的组件，首先在初次渲染时执行 `subscribe(()=>update({}))`，执行一个空对象的`setState`的目的就是为了触发 ui 更新。
这样在对应的组件里面每次更新数据就不会再需要写`setState(reducer(state,{type,payload}))`这么麻烦了，
直接`dispatch({type,payload})`就 OK 啦。

## 精准渲染组件

现在的需要解决的问题在于如果 store 里面的一个内容变化了，那么所有用到 store 的组件都会跟着刷新
~~这好吗？这不好~~ 。所以这次就来实现只渲染数据变化的部分，
如果未发生变化的话就不更新。

首先在 store 的 state 上加一个不会改变的数据`team`，然后把它放到 son 组件中：

```jsx
state: {
    user: { name: 'frank', age: 18 },
    team: { name: 'A-team' }
}
  // son
<section>son team:{stor.team.name}</section>
```

**目标就是当 store 里面只有 user 改变时，son 并不会跟着刷新**

改造当然要放到`connect()`里面啦,先让它接受一个`selector`来让组件选择接受哪些数据：

```jsx
export const connect = (selector) => (Compenent) => {
  return (props) => {
    const { state, setState, subscribe } = useContext(appContext);
    const [, update] = useState({});
    // 这里选择传入的state，默认为整个store
    const data = selector ? selector(state) : { state };
    const dispach = (action) => {
      setState(reducer(state, action));
    };
    useEffect(() => {
      const clean = store.subscribe(() => {
        update({});
      });
      return clean;
    }, [selector]);
    return <Compenent {...props} {...data} dispach={dispach} />;
  };
};
```

然后在所有用到`connect`的地方改造一下:

```jsx
const User = connect()(({ state }) => {
  console.log("user");
  return <div>User:{state.user.name}</div>;
});
const Son = connect((state) => {
  return { team: state.team };
})(({ team }) => {
  console.log("son");
  return <section>son team:{team.name}</section>;
});
```

ok，目前只是做到了**选择数据**，接下来就可以尝试实现精准渲染啦。

其实核心就是对比，也就是在`connect`中 update 的时候，对比一下选择的数据改变了没即可。

```jsx
// 判断数据是否发生变化
const changed = (one, two) => {
    for (let k in one) {
        if (one[k] !== two[k]) {
            return  true
        }
    }
    return false
}
export const connect = (selector) => (Compenent) => {
  return (props) => {
    ...
    const data = selector ? selector(state) : { state }
    useEffect(() => {
        return subscribe(() => {
            const newData = selector ? selector(store.state) : {state: store.state}
            if (changed(data, newData)) {
                update({})
            }
        })
    }, [selector])
    return <Compenent {...props} {...data} dispach={dispach} />
  }
}
```

这样的话，当改变`user.name`时，son 因为并没有用到 user，所以也不会跟着刷新，
这样就实现了精准渲染。

和 select state 一样，还需要用相同的方法加入 select dispatcher，即选择更新的内容，最终的用法应该是这样的

```jsx
// connectToUser.jsx

import { connect } from "../redux";
const userSelector = (state) => {
  return { user: state.user };
};
const userDispatch = (dispatch) => {
  return {
    updateUser: (attrs) => dispatch({ type: "updateUser", payload: attrs }),
  };
};
export const connectToUser = connect(userSelector, userDispatch);
```

在 connect 内部实现：

```jsx
 ...
const dispatcher = dispatcherSelector ? dispatcherSelector(dispatch) : {dispatch}
 ...
return <Component {...props} {...data} {...dispatcher} />
```

然后在组件中使用的时候，就可以直接使用`updateUser`，同时所有用到对应`store`的地方都会更新

```jsx
const UserModifier = connectToUser(({ updateUser, user }) => {
  console.log("usermocidifer");
  const onChange = (e) => {
    updateUser({ name: e.target.value }); // here
  };
  return (
    <div>
      <input value={user.name} onChange={onChange} />
    </div>
  );
});
```

这样就算基本完成了 redux 的核心功能，其中最主要的还是`connect`的实现，放`connect`的最终代码：

```jsx
export const connect = (StateSelector, dispatcherSelector) => (Component) => {
  return (props) => {
    const dispatch = (action) => {
      setState(store.reducer(state, action));
    };
    const { state, setState, subscribe } = useContext(appContext);
    const [, update] = useState({});
    const data = StateSelector ? StateSelector(state) : { state };
    const dispatcher = dispatcherSelector
      ? dispatcherSelector(dispatch)
      : { dispatch };
    useEffect(() => {
      return subscribe(() => {
        const newData = StateSelector
          ? StateSelector(store.state)
          : { state: store.state };
        if (changed(data, newData)) {
          update({});
        }
      });
    }, [StateSelector]);
    return <Component {...props} {...data} {...dispatcher} />;
  };
};
```

只要理解它就可以弄明白 redux 的思想是什么样子的了
