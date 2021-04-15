---
title: 通过context实现redux
date: 2021-04-13 14:59:58
tags: react
---

先看页面：

![1](/images/redux-1.png)

目前的实现非常简单，直接看代码：

```
// App.jsx
import React from 'react'
import {connect,appContext,store} from './redux'

export const App = () => {
  console.log('app');
  return (
    <appContext.Provider value={store}> //store里面存的就是全局的数据
      <Grandfather /> // 只是单纯的展示store里面的数据
      <Father /> // 可以修改store中的数据
      <Son /> // 完全静态的组件
    </appContext.Provider>
  )
}

const Grandfather = () => {
  console.log('grandfather')
  return (<section>Grandfather<User /></section>)
}
const User = connect(({state}) => {
  console.log('user')
  return <div>User:{state.user.name}</div>
})

const Father = () => {
  console.log('father')
  return (<section>father <UserModifier /></section>)
}
const Son = () => {
  console.log('son')
  return (<section>son</section>)
}

const _UserModifier = ({dispach,state}) => {
  console.log('usermocidifer');
  const onChange = (e) => {
    dispach({type:'updateUser',payload:{name:e.target.value}})
  }
  return <div>
    <input value={state.user.name}
      onChange={onChange}/>
  </div>
}
const UserModifier = connect(_UserModifier)
```

`store`,`dispatch`和`connect`都写在 redux.jsx 中

为了避免不必要的渲染，所以在 store 里面加上了订阅模式,
订阅时需要传入更新的函数（setState）使对应的 ui 更新，在`store.setState`的时候就会执行 listener list 中的函数，
在对应组件销毁时时会清除 listener 里面的更新函数

`reducer`没啥好说的，就是更新`store.state`里面的数据

```
// reudx.jsx

import React ,{useState,useContext , useEffect} from 'react'

export const appContext = React.createContext(null)

export const store = {
  state:{
    user: {name: 'frank', age: 18}
  },
  setState:(newState)=>{
    store.state = newState
    store.linsteners.map(fn=>fn(store.state))
  },
  linsteners:[],
  subscribe:(fn)=>{
    store.linsteners.push(fn)
    return ()=>{
      const newList = store.linsteners.filter(f=>JSON.stringify(f)!==JSON.stringify(fn))
      store.linsteners = newList
    }
  }
}
// reducer 即为更新state数据

const reducer = (state,{type,payload})=>{
  if(type === 'updateUser'){
    return {
      ...state,
      user:{
        ...state.user,
        ...payload
      }
    }
  }else{
    return state
  }
}

export const connect = (Compenent)=>{
  return (props)=>{
    const {state, setState,subscribe} = useContext(appContext)
    const [,update] = useState({})
    const dispach = (action)=>{
      setState(reducer(state,action))
    }
    useEffect(()=>{
      const clean = subscribe(()=>update({}))
      return ()=>{
        clean()
      }
    },[])
    return <Compenent {...props} dispach={dispach} state={state} />
  }
}

```

`connect`用于连接组件,返回一个被连接到 store 的组件，首先在初次渲染时执行 `subscribe(()=>update({}))`，执行一个空对象的`setState`的目的就是为了触发 ui 更新。
这样在对应的组件里面每次更新数据就不会再需要写`setState(reducer(state,{type,payload}))`这么麻烦了，
直接`dispatch({type,payload})`就 OK 啦。
