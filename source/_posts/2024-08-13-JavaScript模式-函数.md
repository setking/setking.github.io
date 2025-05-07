---
title: 函数模式
author: cotes
date: 2024-08-13 11:33:00 +0800
categories: [学习]
tags: [学习]
pin: true
math: true
mermaid: true
---



### JavaScript函数特点

1 、函数是第一类对象，表现如下：

				- 函数可以在运行时动态创建，也可以在程序执行过程中创建
				- 函数可以分配给变量，可以将函数引用复制到其他变量，可以被扩展，除少数情况之外，还可以被删除
				- 可以作为参数传递，也可以由其他函数返回
				- 函数可以有自己的属性和方法

2、 函数拥有作用域

### 函数回调模式

函数都是对象，所有它们可以作为参数传递给其它函数

示例：

```
function foo(callback) {
    callback()
}

function bar() {

}
foo(bar)
```

> bar作为参数传递给foo是不带括号的。括号表示要执行函数，这里我们只需要传递该函数的应用，让foo在需要的时候执行bar（返回以后调用）

### 自定义函数

函数可以动态定义，也可以分配给变量。如果创建了一个新函数并且将其分配包存了另外函数的同一个变量，那么就以一个新函数覆盖了旧函数。某种程度上，回收了旧函数指针以指向一个新函数。这一切都发生在函数体内部。

示例：

```
let foo = function () {
    console.log("foo!")
    foo = function () {
        console.log("Double foo!")
    }
}

foo() //foo!
foo() //double foo!
```

> 这种模式一般用与初始化工作，仅需要执行一次，此模式可以显著提升性能，减少工作量。该模式的缺点在于，它重新定义自身已添加到原始函数的任何属性都会丢失。而且，如果分配给不同的变量或者以对象方法使用，重新定义将永远不会发生，并将执行原始函数体

### 即时函数

即时函数模式是一种可以支持在定义函数后立即执行该函数的语法。这种模式为初始化代码提供了一个作用域沙箱

示例：

```
(function () {
	console.log('watch!')
} ())
```

第二种写法：

```
(function () {
	console.log('watch!')
})()
```



该模式由以下几部分组成：

	- 可以使用函数表达式定义一个函数（函数声明无法达到这个效果）
	- 末尾添加一组括号，表示该函数立即执行
	- 将整个函数包装在括号里（在不将该函数分配给变量的情况下）

#### 即时函数的参数

示例：

```
(function (global) {
	// 可以通过global访问全局变量
}(this))
```

> 一般不建议传递多个参数，这样会造成阅读负担

即时函数可以包装想要执行且不会在后台留下全局变量的代码。定义的所有变量用于自调用函数的局部变量，并且不用担心全局变量被污染

### 函数的属性--备忘模式

函数是对象，因此它具有属性。可以在任何时候将自定义属性添加到你的函数中。自定义属性其中一个用例是缓存函数结果（也被称为备忘）。

示例：

```
function foo(param) {
	if(!foo.cache[param]) {
		var result = {}
		foo.cache[param] = result;
	}
	return foo.cache[param]
}
// 缓存存储
foo.cache = {}
```



### Curry

`apply()`：` apply()`带有两个参数：第一个参数为将要绑定到该函数内部this的一个对象，而第二个参数是一个数组或多个参数变量，这些参数将变成可用于该函数内部的类似数组的arguments对象。如果第一个参数为null，那么this将指向全局对象，此时得到的结果就恰好如同当调用一个非指定对象时的方法。

`call()`：`call()`当仅有一个参数时，可以避免创建只有一个元素的数组的工作

```
function foo(args) {
    console.log(args)
}
// 第二种更有效率，节省了一个数组
foo.apply(null, ["bar"]) // bar
foo.call(null, "bar") // bar
```



当函数是一个对象的方法时，不能传递null引用。这种情况下，这里的对象将成为`apply()`的第一个参数：

```
let bar = {
    foo: function (args) {
        console.log(args)
    }
}

bar.foo("foo") // foo
foo.apply(bar, ["foo"]) // foo
```

#### Curry化

示例：

```
// curry化的add()函数
// 接受部分参数列表
function add(x, y) {
    let oldX = x, oldY = y;
    if (typeof oldY === 'undefined') {
        return function (newY) {
            return oldX + newY
        }
    }
    //完全应用
    return x + y
}	

add(3)(4) // 7
```

优化：

```
// curry化的add()函数
// 接受部分参数列表
function add(x, y) {
    if (typeof y === 'undefined') {
        return function (y) {
            return x + y
        }
    }
    //完全应用
    return x + y
}

add(3)(4) // 7
```

通用curry化函数：

```
// 通用curry化示例

function curry(fn) {
    let slice = Array.prototype.slice,
        storages = slice.call(arguments, 1)
    return function () {
        let newArgs = slice.call(arguments),
            args = storages.concat(newArgs)

        return fn.apply(null, args)
    }
}
```

