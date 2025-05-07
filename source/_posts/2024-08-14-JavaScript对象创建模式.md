---
title: 对象创建模式
author: setKing
date: 2024-08-14 11:33:00 +0800
categories: [学习]
tags: [学习]
pin: true
math: true
mermaid: true
---

### 通用命名空间函数

示例：

```
var MYAPP = MYAPP || {};

MYAPP.namespace = (ns_string) => {
    var parts = ns_string.split('.'), parent = MYAPP, i
    if (parts[0] === "MYAPP") {
        parts = parts.slice(1)
    }
    for (i = 0; i < parts.length; i++) {
        if (typeof parent[parts[i]] == "undefined") {
            parent[parts[i]] = {}
        }
        parent = parent[parts[i]]
    }
    return parent
}

var module2 = MYAPP.namespace("MYAPP.modules.module2")
```

### 声明依赖关系

js 库通常是模块化且依据命名空间组织的，这使得能够仅包含所需的模块。

显示的依赖声明能明确的表示需要的特定脚本已经包含在了该页面中。在函数顶部的前期声明可以很容易的发现和解析依赖。在 js 中解析局部变量（比如 DOM）的速度总是要比解析全局变量要快，甚至比解析全局变量的嵌套属性还要快。

#### 原型和私有性

将私有成员与构造函数一起使用时，其中一个缺点在于每次调用构造函数以创建对象时，这些私有成员都会被重新创建。

##### 私有成员

虽然 js 没有用于私有成员的特殊语法，但是可以使用闭包来实现这个功能。构造函数创建一个闭包，闭包范围内的任意变量都不会暴露到构造函数之外的代码。不过这些私有变量任然可以用于共有方法：定义在构造函数中，作为对象的一部分暴露给外部的方法。

示例：

```
function foo() {
	// 私有成员
	var a = 1
	// 共有函数
	this.getData = function() {
		return a
	}
}
var bar = new foo()
console.log(bar.a) // undfined
console.log(bar.getData) // 1
```

> 尽量不要传递保持私有性的对象和数组的引用

#### 将私有方法揭示为公共方法

示例：

```
var useArray;
(function () {
    var str = "[object Array]",
        toString = Object.prototype.toString
    function isArray(a) {
        return toString.call(a) === str
    }
    function indexOf(haystack, needle) {
        var i = 0,
            max = haystack.length
        for (; i < max; i++) {
            if (haystack[i] === needle) {
                return i
            }
        }
        return -1
    }
    // 提供公共访问功能
    useArray = {
        isArray: isArray,
        indexOf: indexOf,
        inArray: indexOf
    }
}())
```

### 模块模式

模块模式由以下模式组合而来：

- 命名空间
- 即时函数
- 私有和特权成员
- 声明依赖

该模式的第一步是建立一个命名空间

```
MYAPP.namespace("MYAPP.util.array")
```

接下来定义模块（对于需要保持私有性的情况，可以提供一个具有私有作用域的即时函数，该函数返回一个对象，即具有公共接口的实际模块，可以通过这些接口来使用这些模块）

```
MYAPP.util.array = (function() {
        return {
			// todo...
        }
}())
```

然后向接口提供一些方法：

```
MYAPP.util.array = (function () {
	return {
		inArray: function(needle, haystack) {
			// ...
		},
		isArray: function(a) {
			// ...
		}
	}
} ())
```

> 通过即时函数提供的私有作用域，可以根据需要声明一些私有属性和方法

### 揭示模块模式

示例：

```
MYAPP.util.array = (function () {
		// 私有属性
		var array_string = "[object Array]",
			ops = Object.prototype.toString,
		// 私有方法
		inArray: function(needle, haystack) {
			// ...
		},
		isArray: function(a) {
			// ...
		}
		// 揭示公有API
		return {
			isArray: isArray,
			indexOf: inArray
		}
} ())
```

### 沙箱模式

沙箱模式提供了一个可用于模块运行的环境，且不会对其他模块和个人沙箱造成任何影响

#### 全局构造函数

示例：

```
new sandbox(function (box) {
	// code
})
```

> `sandbox()` 构造函数可以接受一个额外配置参数（或多个参数），该参数指定了对象实例所需要的模块名。沙箱模式可以嵌套，并且两者不会相互干扰

### 静态成员

静态属性和方法就是从一个实例到另外一个实例都不会发生改变的属性和方法。

#### 公有静态成员

示例：

```
// 构造函数
var foo = function () { }
// 静态方法
foo.isFoo = function () {
    return "foo"
}
// 向原型添加普通方法
foo.prototype.setBar = function (bar) {
    this.bar = bar
}
// 调用静态方法
foo.isFoo() // foo
// 创建示例并调用
var newBar = new foo()
newBar.setBar(30) // 30
```

> 可以很方便的实现静态方法与实例一起工作，在原型中添加一个新方法指向原始静态方法的名字： `foo.prototype.isFoo = foo.isFoo`。不过这种情况需要注意 this 指向问题，执行`foo.isFoo()`时，`isFoo()`内部的 this 将会指向 foo 构造函数

#### 私有静态成员

私有静态成员有如下属性：

- 以同一个构造函数创建的所有对象共享该成员
- 构造函数外部不可访问该成员

示例：

```
var foo = (function () {
    // 静态属性/变量
    var counter = 0
    // 返回
    // 构造函数新的实现
    return function () {
        console.log(counter += 1)
    }
}())
```
