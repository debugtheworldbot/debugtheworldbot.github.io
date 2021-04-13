---
title: 通过context实现redux
date: 2021-04-13 14:59:58
tags: react
---

先看页面：
![hh](/images/redux-1.png)
目前的实现非常简单，直接看代码：


```
// App.jsx

export const App = () => {
  console.log('app');
  return (
    <appContext.Provider value={store}> //store里面存的就是全局的数据 
      <Grandfather /> // 只是单纯的展示store里面的数据
      <Father /> // 可以修改store数据
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
