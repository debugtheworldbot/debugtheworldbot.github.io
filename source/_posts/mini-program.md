---
title: 小程序踩坑记录
date: 2021-04-15 14:40:19
tags:
---

### html

类似于隐私协议的应用html文件（dangerouslySetInnerHTML），无法通过标签名来修改样式，但是可以通过正则给所有的标签加上class

```tsx
export function addRichTextClass(text:string){
  // 去掉标签里面的style和class，加上统一的class，例如p_class
  const rmStyle =(tag:string):[RegExp,string] =>
    [new RegExp(`<${tag}([\\s\\w\"=\\/\\.:;]+)((?:(style=\"[^\"]+\")))`,"ig"),`<${tag}`];
  const rmClass =(tag:string):[RegExp,string] =>
    [new RegExp(`<${tag}([\\s\\w\"=\\/\\.:;]+)((?:(class=\"[^\"]+\")))`,"ig"),`<${tag}`]
  const addClass =(tag:string):[RegExp,string] =>
    [new RegExp(`<${tag}>`,"ig"),`<${tag} class="${tag}_class">`]

  return text
      .replace(...rmStyle('p'))
      .replace(...rmClass('p'))
      .replace(...addClass('p'))

      .replace(...rmStyle('span'))
      .replace(...rmClass('span'))
      .replace(...addClass('span'))

      .replace(...rmStyle('section'))
      .replace(...rmClass('section'))
      .replace(...addClass('section'))
}

usage
<View className='protocol-wrapper'>
 <RichText nodes={addRichTextClass(protocolText)} />
</View>
```

### [Please do not register multiple Pages](https://github.com/NervJS/taro/issues/6084#)

https://github.com/NervJS/taro/issues/6084 

简单来说，在`app.config.js`的`pages`中配置了某个页面，然后又在某处代码中import了这个页面里面的函数，就会出现这个错误。

原因：这样的行为是不推荐的，因为页面文件会被一个特殊的 loader 处理[1]，如果同一个页面多次运行这个 loader 可能会出现意想不到的情况出现。

如果确实需要在其他地方引用页面组件，我们推荐把页面组件的逻辑抽象为一个单独的普通组件，然后在页面组件中直接渲染或继承这个普通组件，其它的地方也可以正常引入并使用这个普通组件。

之后的版本在编译时检测这样的行为会直接报错并引导到此 issue。

[1] 具体来说就是创造一个小程序 `Page()` 函数能接受的对象，并调用 `Page()` 函数，由于 `Page()` 必须调用才能创建页面，而 Taro 没办法控制 `Page()` 调用之后的结果，因此这个 loader 是做不到幂等的。



### jsx的map问题

```jsx
{[['a']].map(b=>
  <View>
    {b.map(c=><View>{c}</View>)}
  </View>)}
```

![image-20210323170914216](/Users/g2/Library/Application Support/typora-user-images/image-20210323170914216.png)

其实就是需要加key，这报错看的一头雾水

### 全面屏适配问题

![image-20210324143954710](/Users/g2/Library/Application Support/typora-user-images/image-20210324143954710.png)

像这种底部固定的按钮，底部需要留一个安全区 

很简单 加上` env(safe-area-inset-bottom)`就可以了

![image-20210324144056584](/Users/g2/Library/Application Support/typora-user-images/image-20210324144056584.png)

比如这个是136px 那么这部分的height为 `height: calc( 136px + env(safe-area-inset-bottom));`

参考：https://blog.csdn.net/qq_42354773/article/details/81018615



### Taro.navigateBack() 无法向回退的页面传参

小程序的几个导航api,都可以通过 **url** 给对应的页面传参。而 Taro.navigateBack({delta}), 只接受一个delta（返回的页面数）参数。但是有时候确确实实有向回退页面传参数的情况，这时候就只能通过mobx之类的来处理了。