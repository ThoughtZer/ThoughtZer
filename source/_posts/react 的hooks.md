---
title: React 的 Hook 们
index_img: /img/react/react.jpeg
tags: 框架
date: 2019-09-26 20:20:30
---

React 在 v16.8 版本中正式推出了 Hooks，为函数式组件带来了“春天”。
<!-- more -->

本文简单介绍几个在项目中使用过的hook。

### useState

以往class写法定义状态需要挂载到```this.state```上，在使用函数式组件的时候不能定义状态。useState让不可能变成了可能。

```js
// 声明 value 以及 设置 value 的方法 setValue
const [value, setValue] = useState('initVal');

// 之后在JSX以及组件内部就可以使用变量以及设置方法
```

### useEffect

直译中文就是副作用钩子。默认情况下赋值给 useEffect 的函数会在组件渲染之后执行。当然也可以传递参数执行在某一些值变化之后执行，有点类似于 Vue 的 watch，(当然Vue的3.0版本可能也会更改为Effect)~

```js
// 在每一次渲染之后执行
useEffect(() => {
  // do something
});

// 在第二个参数数组中的数值变化后就执行
useEffect(() => {
  // when variable1 or variable2 was changed, do something
}, [variable1, variable2, ...]);

// 组件卸载时需要清除 effect 中的副作用
useEffect(() => {
  const timer = setTimeout(() => {}, 1000);
  return () => {
    clearTimeout(timer)
  }
}, []);
```

useEffect 的执行时机和 class 写法的```componentDidMount```、```componentDidUpdate```有很大契合，但是也有不同的地方，官方给出了详细的解释，[传送门](https://reactjs.org/docs/hooks-reference.html#timing-of-effects)。

### useCallback

使用 useCallback 传递一个回调函数，可以返回一个被记忆的回调函数。且只会在依赖项发生变化的时候，重新更新回调函数，当我们在传递回调给子组件的时候，这将大有用处: 即使本次父组件由于依赖的更新而重新执行了 ```render```，那么这个回调的 props 不会重新生成，那么在进行了引用相等性判断了的子组件接收这个回调 props 不会再次渲染

```js
// 如果没有特别的需要，推荐都使用 useCallback 处理函数
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### useMemo

使用 useMemo 传递一个有返回值的回调函数，则回调函数的返回值会是一个被记忆的值，同 useCallback 返回一个被记忆的函数一致。

```js
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
// 上面等同于
const memoizedCallback = useMemo(() => () =>{
  doSomething(a, b);
}, [a, b]);
// 当然不仅限于函数
const memoizedValue = useMemo(() => {
  doSomething(a, b);
  return {
    ...
  }
}, [a, b]);
```

官方推荐: 先编写在没有 useMemo 的情况下也可以执行的代码 —— 之后再在你的代码中添加 useMemo。因为缓存太多的内容可能会造成危险，所以当达到某一临界值的时候，可能就不能记忆很久之前的值，需要重新计算。

### useRef

也许 ref 真的是毒药，毕竟像 React 这类 UI 库，不推荐我们再继续操作 DOM。但是这并不能囊括所有的场景，我们还是需要 ref 的，并且可能不仅仅是为了操作 DOM，比如需要在父组件执行子组件的方法时。需要注意的是当前 react 已经把获取到的实例放到了 ref 的 current 属性下。

```js
// 父组件
const Parent = () => {
  const domRef = useRef(null)
  return (
    <>
      <Child ref={domRef} />
    </>
  );
}
```

### useImperativeHandle

在说 useRef 的时候说到了需要在父组件中执行子组件的方法。那么就可能需要 useImperativeHandle 来帮忙了。useImperativeHandle 会在子组件内部通过接收在父组件中传递过来的 ref，绑定一些可执行方法，供父组件使用。需要注意的是 useImperativeHandle 需要 forwardRef 函数的帮助。

```js
// 父组件
const Parent = () => {
  const ChildRef = useRef(null)
  // 父组件就可以使用 ChildRef 调用子组件方法
  ChildRef.current.method_1();
  return (
    <>
      <Child ref={ChildRef} />
    </>
  );
}
// 子组件
const Child = forwardRef((props, ref) => {
  useImperativeHandle(ref, () => {
    return {
      method_1: () => {},
      method_2: () => {},
    };
  });
  return (
    ...
  );
})
```

个人感觉 useImperativeHandle 是一个有意思的 hook，让 Vue 的开发者可能会更喜欢~

### useContext 和 useReducer

useContext 和 useReducer 个人使用的情景不是很多，不在讲述~ 但是网上有人说可以代替 redux 的位置，emmmmmm，作为一个 react 的小白开发者，暂时觉得还是 redux 的全家桶在 react 中使用着更舒服、更流畅。
