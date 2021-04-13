---
title: 通过context实现redux
date: 2021-04-13 14:59:58
tags: react
---

先看页面：

![](/images/redux-1.png)

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

`store`,`dispatch`和`connect`都写在redux.jsx中

为了避免不必要的渲染，所以在store里面加上了订阅模式

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
