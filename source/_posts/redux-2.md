---
title: 通过context实现redux(2)
date: 2021-04-13 14:59:58
tags: react
sticky: 1
---

## 精准渲染组件

现在的需要解决的问题在于如果store里面的一个内容变化了，那么所有用到store的组件都会跟着刷新
~~这好吗？这不好~~ 。所以这次就来实现只渲染数据变化的部分，
如果未发生变化的话就不更新。

首先在store的state上加一个不会改变的数据`team`，然后把它放到son组件中：

```jsx
state: {
    user: { name: 'frank', age: 18 },
    team: { name: 'A-team' }
}
  // son
<section>son team:{stor.team.name}</section>
```

**目标就是当store里面只有user改变时，son并不会跟着刷新**

改造当然要放到`connect()`里面啦,先让它接受一个`selector`来让组件选择接受哪些数据：

```jsx
export const connect = (selector) => (Compenent) => {
  return (props) => {
    const { state, setState, subscribe } = useContext(appContext)
    const [, update] = useState({})
      // 这里选择传入的state，默认为整个store
    const data = selector ? selector(state) : { state }
    const dispach = (action) => {
      setState(reducer(state, action))
    }
    useEffect(() => {
      const clean = store.subscribe(() => {
         update({})
      })
      return clean
    }, [selector])
    return <Compenent {...props} {...data} dispach={dispach} />
  }
}
```

然后在所有用到`connect`的地方改造一下:
```jsx
const User = connect()(({ state }) => {
  console.log('user')
  return <div>User:{state.user.name}</div>
})
const Son = connect(state => {
    return { team: state.team }
})(({ team }) => {
    console.log('son')
    return (<section>son team:{team.name}</section>)
})
```

ok，目前只是做到了**选择数据**，接下来就可以尝试实现精准渲染啦。

其实核心就是对比，也就是在`connect`中update的时候，对比一下选择的数据改变了没即可。

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

这样的话，当改变`user.name`时，son因为并没有用到user，所以也不会跟着刷新，
这样就实现了精准渲染。


