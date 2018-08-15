---
title: 用airbnb的JavaScript规则来提高代码质量
date: 2018-08-13 12:59:09
tags: 
- eslint
- airbnb
- 代码规范
---

# 前言
ESLint最初是由Nicholas C. Zakas 于2013年6月创建的开源项目。它的目标是提供一个插件化的javascript代码检测工具。eslint不仅能统一团队代码，而且解决代码潜在的bug。
虽然eslint是支持自己配置规则的，但是怕自己学艺未精配置出怪异规则集，故还是使用现有的`eslint-config`。现在比较流行的eslint规则集有[standard](https://standardjs.com)、[Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)、[Google JavaScript Style Guide](https://google.github.io/styleguide/jsguide.html#features-array-literals)。本文主要围绕`Airbnb JavaScript Style Guide`展开。
通过阅读规则`Airbnb JavaScript Style Guide`，我学到了很多编写JavaScript技巧。
我把它们做了一下筛选（事例代码均来自[Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)）。

# 规则集（精选）
## 引用（References）
### 1. `prefer-const` `no-const-assign`
> Why? This ensures that you can’t reassign your references, which can lead to bugs and difficult to comprehend code.

尽量使用const定义常量。对于对象，const能确保引用不变。实在是需要变化的数据才使用let。

```javascript
// bad
var a = 1;
var b = 2;

// good
const a = 1;
const b = 2;
```

### 2. `no-var`
> Why? let is block-scoped rather than function-scoped like var.

为什么不建议使用var？
1. let、const关键词提供了块级作用域。
2. 防止变量提升。
3. let、const暂存死区的错误，防止变量重复定义。

```javascript
// bad
var count = 1;
if (true) {
  count += 1;
}

// good, use the let.
let count = 1;
if (true) {
  count += 1;
}
```

## 对象（Objects）
### 1. 不要直接使用Object.prototype上的方法。
> Why? These methods may be shadowed by properties on the object in question - consider { hasOwnProperty: false } - or, the object may be a null object (Object.create(null)).

1. 如果对象中有一个属性和Object.prototype上的方法重名了，该重名属性是会覆盖掉原型上的方法。
2. 如果对象为空，null是没有Object.prototype上的方法的。浏览器通常会报`"Cannot read property 'hasOwnProperty' of null"`

```javascript
// bad
console.log(object.hasOwnProperty(key));

// good
console.log(Object.prototype.hasOwnProperty.call(object, key));

// best
const has = Object.prototype.hasOwnProperty; // cache the lookup once, in module scope.
/* or */
import has from 'has'; // https://www.npmjs.com/package/has
// ...
console.log(has.call(object, key));
```

## 数组(Arrays)
### 1. 数组复制
使用扩展运算符(`...`)进行数组复制。从下面代码可以看到使用扩展运算符更加简便。

```javascript
// bad
const len = items.length;
const itemsCopy = [];
let i;

for (i = 0; i < len; i += 1) {
  itemsCopy[i] = items[i];
}

// good
const itemsCopy = [...items];
```

### 2. 可迭代对象使用扩展运算符转换为数组
这里需要注意的一点，必须是`可迭代`的对象。
> 为了变成可迭代对象， 一个对象必须实现 @@iterator 方法, 意思是这个对象（或者它原型链 prototype chain 上的某个对象）必须有一个名字是 Symbol.iterator 的属性

常用的可迭代对象有，`arguments`，`NodeList`等常用的数据结构。

Array.from固然可以把类数组对象转换为数组。
从简而言，扩展运算符更加优雅。

```javascript
const foo = document.querySelectorAll('.foo');

// bad 不直观
const nodes = Array.prototype.slice.call(foo);

// good 可以
const nodes = Array.from(foo);

// best 简洁
const nodes = [...foo];
```

## 解构（Destructuring）
### `prefer-destructuring`
>ES6 允许按照一定模式，从数组和对象中提取值，对变量进行赋值，这被称为解构（Destructuring）。

使用解构赋值，更加直观。解构赋值能减少为属性创建临时变量。

```javascript
// bad
function getFullName(user) {
  const firstName = user.firstName;
  const lastName = user.lastName;

  return `${firstName} ${lastName}`;
}

// good
function getFullName(user) {
  const { firstName, lastName } = user;
  return `${firstName} ${lastName}`;
}

// best
function getFullName({ firstName, lastName }) {
  return `${firstName} ${lastName}`;
}

const arr = [1, 2, 3, 4];

// bad
const first = arr[0];
const second = arr[1];

// good
const [first, second] = arr;
```

## 箭头函数
### prefer-arrow-callback
回调函数尽量使用箭头函数。
1. 箭头函数不会绑定this。在箭头函数出来之前，每个函数都会有自己的this值。箭头函数不会创建自己的this,它只会从自己的作用域链的上一层继承this。
2. 箭头函数提供更简短的函数。

不过在使用箭头函数的时候需要注意以下几点：
1. 由于 箭头函数没有自己的this指针，通过 call() 或 apply() 方法调用一个函数时，只能传递参数，他们的第一个参数会被忽略。
2. 箭头函数不绑定`Arguments`对象。箭头函数通常使用rest参数。
3. 箭头函数不能用作构造器，和 new一起用会抛出错误。
4. 箭头函数没有prototype属性。

```javascript
// bad
[1, 2, 3].map(function (x) {
  const y = x + 1;
  return x * y;
});

// good
[1, 2, 3].map((x) => {
  const y = x + 1;
  return x * y;
});
```

## 比较操作符和等值(Comparison Operators & Equality)
### 1. 条件表达式真值问题
* 所有的Objects（空数组也是Object，所以也是true）都是true
* Undefined为false
* Null为false
* Booleans值：true为true，false为false
* Numbers值+0, -0, NaN时为false, 其它数值是true
* Strings值''（空字符串）为false, 其它非空字符串true

更多可以查看[Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy)

### 2. Boolean值尽量使用简约写法，string和number值要显示比较。
```javascript
// bad
if (isValid === true) {
  // ...
}

// good
if (isValid) {
  // ...
}

// bad
if (name) {
  // ...
}

// good
if (name !== '') {
  // ...
}

// bad
if (collection.length) {
  // ...
}

// good
if (collection.length > 0) {
  // ...
}
```

### 3.no-case-declarations
每一个case和default语句必须要使用花括号。
词法作用域覆盖整个switch语句。case中可能重复定义或者使用某个变量名称。如果使用了`let`或者`const`会触发`暂时性死区`，运行就会报重复定义变量的错误。使用花括号`{}`可以提供块级作用域，避免触发`暂时性死区`。

```javascript
switch (foo) {
  case 1: {
    let x = 1;
    break;
  }
  case 2: {
    const y = 2;
    break;
  }
  case 3: {
    function f() {
      // ...
    }
    break;
  }
  default: {
    class C {}
  }
}
```

### 4.`no-unneeded-ternary`
避免使用不必要的三元表达式。

```javascript
// bad
const foo = a ? a : b;
const bar = c ? true : false;
const baz = c ? false : true;

// good
const foo = a || b;
const bar = !!c;
const baz = !c;
```

逻辑非操作符由一个叹号（!）表示，可以应用于ECMASript中的任何值。无论这个值是什么数据类型，这个操作符都会返回一个布尔值。  
同时使用两个逻辑非操作符，实际上就会模拟Boolean()转型函数的行为。第一个逻辑非操作会基于无论什么操作数返回一个布尔值，而第二个逻辑非操作则对该布尔值求反，于是就得到了这个值真正对应的布尔值。

# 总结
阅读完airbnb的规则后，给我最大的感觉是，太严格了！  
* 能简则简，简约至上
* 无论从变量、参数、循环都可以看出airbnb非常喜欢`immutable rule`(不变原则)

建议大家都去[Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)看一下，真的能学到很多东西！！