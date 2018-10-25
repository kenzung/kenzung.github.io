---
title: 深入浅出redux学习（-）
date: 2018-10-10 20:35:18
tags:
- redux
---

> 前言：  
> 一开始接触redux的时候最令我记住的一句话是：`You Might Not Need Redux`（那我还写这篇文章干嘛？手动滑稽）  
> 回归正题，本文主要是围绕redux的作者Dan的视频，由浅入深了解redux

# redux基础用法
---
## 1. Action
如果需要`改变`state（状态），我们需要使用到Action。Action是一个普通的JavaScript对象描述state变化。action的属性可以自定义，但是必须有一个叫`type`的属性，值是string类型（方便序列化）。每一个action都是对state的一个（minimal change）最小修改，在应用里什么东西发生了变化。  

```javascript
  //创建一个加法器，减法器
  const INCREMENT = 'INCREMENT';
  
  {
    type: INCREMENT
  }

  const DECREMENT = 'DECREMENT';
  
  {
    type: DECREMENT
  }
  
  //-----------------------------

  //比如添加新todo任务
  // action type字符常量
  const ADD_TODO = 'ADD_TODO';

  // ADD_TODO action
  {
    type: ADD_TODO,
    text: 'Learn Redux'
  }
```


## 2. 纯函数和非纯函数
什么是纯函数？
> 纯函数就是没有副作用的函数，不包含数据库查询、网络请求等操作。只要输入的值不变，每次输出都是一样的值。

```javascript
//pure function
function square(x) {
  return x * x;
}

// Impure function
function square(x) {
  updateXInDatabase(x);
  return x * x;
}
```

为什么需要说纯函数？  
因为在redux中，有些函数必须是纯函数，比如reducer，所以在编码的时候有必要注意。

## 3. reducer
**注意！**reducer必须是纯函数，不能更改state，每次都必须返回一个新的state。  
reducer，其实就是描述旧状态（previous state）如何转换为当前状态（current state）的函数。  

这里使用的是计数器例子，我们通过`action`的`type`

```javascript
const counter = (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1;
    case 'DECREMENT':
      return state - 1;
    default:
      return state;
  }
}
```
reducer是需要固定的形式的，需要传入2个参数，一个是state，另一个是action。  
我们这里先不要管为什么要这么写，记住这是redux的规定即可。我们可以通过es6的默认参数方式赋予state初始默认值。

## 4. 编写reducer注意点
因为reducer是纯函数，因此需要注意几点：
1. 不能改变参数
2. 不能使用非纯函数

接下来我们一起看一下怎么处理reducer中对数据的处理。  
reducer的state处理通常是两种数据类型：
1. Object
2. Array

### Object
```javascript
// 注意，不能直接改变state中的属性值
state.name = 'ken' // ❌这种对name属性赋值操作是不允许的

// ---------------------------
// 正确✔️做法
// 1，使用Object的assign方法，需要对浏览器做polyfill
return Object.assign({}, state, { name: 'ken' });

// 或者使用es7的解构方法
return {
  ...state,
  name: 'ken',
};
```

### Array
array的操作中，常用操作为：
1. 增（push）
2. 删（splice）
3. 指定位置元素运算操作，如 `array[index]++` `array[index] = array[index] * 2`等操作

*注意*上述3个操作都是会改变原数组，因此需要使用“纯”操作代替这些“非纯”操作。
1. push/pop/shift/unshift
如果需要使用push，我们可以使用concat方法代替。  
concat方法返回一个新数组，且不改变原数组。
```javascript
array.push(x); // ❌
array.concat(x); // ✅
// 或者使用解构方式
[...array, x];
```

2. splice
同理，splice操作也是会改变原数组，我们可以用slice去代替。
```javascript
array.splice(index, 1); // ❌
[...array.slice(0, index),
 ...array.slice(index + 1)]; // ✅
```

3. 指定位置元素运算操作
```javascript
// 如 
array[index]++ // ❌
// 可用以下方式代替 ✅
[...array.slice(0, index),
array[index] + 1,
...array.slice(index + 1)];
```
## 5. createStore
createStore主要是生成redux中最核心的Store对象。`action`描述“发生了什么”，`reducer`是响应`action`并对state进行更新。而`store`可以看成把`action`和`reducer`联系起来的一个对象。

```javascript
import { createStore } from redux;

const store = createStore(counter)

console.log(store.getState());
// output 0

store.dispatch({ type: 'INCREMENT'} );
console.log(store.getState());
// output 1

// ------------------------------
const render = () => {
  document.body.innerText = store.getState();
};
store.subscribe(render);

document.addEventListener('click', () => {
  store.dispatch({ type: 'INCREMENT' });
});
```

`createStore`生成的store对象包含3个方法，分别为`getState`，`dispatch`和`subscribe`。其中`subscribe`函数的返回值是一个函数，用于取消订阅。

* `getState()`返回应用当前的state树。
* `dispatch(action)`分发action。这个是触发`state`改变的**唯一**方法。  
  action是描述应用变化的普通对象。按照约定，action 具有 type 字段来表示它的类型。type 也可被定义为常量或者是从其它模块引入。
* `subscribe(listener)`添加一个变化监听器。listener是一个函数，每当执行dispatch方法时候就会执行listener。  在例子中，每次dispatch一个action，就会触发render函数执行。如果需要解除绑定监听，执行`subscribe`返回的函数即可。

## 6. 实现一个简单版的createStore
我们知道createStore返回的是一个store对象，其中包括`getState`，`dispatch`和`subscribe`方法。  
当然接下来的实现是最简单的实现，去掉了很多参数校验、store enhancer（即中间件）和类RxJS等reactive库支持。有兴趣更深了解的同学可以看看redux的`createStore`源码（ps：我打算下一篇文章写关于redux源码☺️）

```javascript
const CreateStore = (reducers, initState = {}) {
  let currentState = initState;
  let listeners = [];

  // getState就是简单的把当前的state返回给调用者
  const getState = () => currrentState;

  // subscribe：
  // 如果对观察者模式（发布-订阅模式）比较熟悉的同学就会发现，这其实就是做订阅
  // 
  const subscribe = listener => {
    listeners.push(listener);
    return function unsubscribe() {
      listeners = listeners.filter(currentListener => currentListener !== listener)
    };
  };

  // 同理，这个是观察者模式中的发布
  // 接收到action后，通过reducer更新state，并且把listeners执行一遍
  const dispatch = action => {
    reducers(currentState, action);
    listens.forEach(listener => listener());
  }

  // 初始化
  // 让reducers返回他们的初始state
  dispatch({});

  return {
    getState,
    subscribe,
    dispatch,
  };
};
```
我们很简单就实现了`createStore`，当然实际上还是会稍微复杂一点。

## 7. combineReducers
从上述我们实现的`createStore`源码可以看出，传入的reducer只有`一个`。  
真实的应用中reducer是多个的，因此我们需要把reducer组合起来。`combineReducers`就是帮我们完成这个任务。

```javascript
import { combineReducers } from 'redux';

const todosReducer = (state = {}, action) => {
  switch (action.type) {
    ...
  }
};

const visibilityFilterReducer = (state = {}, action) => {
  switch (action.type) {
    ...
  }
};

const reducer = combineReducers({
  todos: todosReducer,
  visibilityFilter: visibilityFilterReducer,
});

const store = createStore(reducer, {});
```

代码主要完成了把todosReducer和visibilityFilterReducer合并为一个reducer；todos和visibilityFilter分别是两个变量接收两个reducer处理的结果。

其实我们可以这么理解
```javascript
const todosAndVisibilityFilterReducer = (state = {
  todos: {},
  visibilityFilter: {}
}, action) => {
  switch (action.type) {
    ...
    // 合并todosReducer和visibilityFilterReducer的case即可
  }
}

const store = createStore(todosAndVisibilityFilterReducer, {});
```

当然真实项目不能这么搞，既然redux提供了combineReducers，我们就应该使用。


## 8. 实现一个简单版的combineReducers
和createStore一样，我们写一个简单版本的`combineReducers`。
```javascript
//可以理解combineReducers就是一个reducer，因此其返回值是一个可以传入(state, action)的函数
const combineReducers = (reducers = {}) => {
  return (state = {}, action) => {
    // 和普通的reducer一样，返回一个新的state作为返回值
    // 此处选择Array.reducer方法更加简洁；forEach还要创建一个临时变量保存新值。
    return Object.keys(reducers).reduce((nextState, key) => {
      // 这里的key可以套入todos/visibilityFilter
      // 执行每个reducer方法，并取得其返回的state值，放入nextState中
      nextState[key] = reducers[key](state[key], action);
      return nextState;
    });
  }
};
```


# 参考
1. [redux官网](https://redux.js.org/)
2. [Dan的Getting Started With Redux系列课程（非常赞，5星级推荐）](https://egghead.io/series/getting-started-with-redux)