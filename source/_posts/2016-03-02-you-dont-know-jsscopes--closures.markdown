---
layout: post
keywords: blog
description: blog
title: "You don't know js(scopes & Closures)"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2016-03-02 17:53:30
---

#作用域和闭包

最近读了You don't know js的`Scope & Closures`章节，记录一下学到的。

##编译简介

先简单介绍一下一般语言的编译过程：

1. 标记解释/词法分析

	把一段字符串的程序解析成几个有意义的部分。比如`var a = 2;`在这一步中会被解释成`var`, `a`, `=`, `2`和`;`。这两个词的具体意思是什么呢？或者说区别是什么，举个例子，这里决定`a`是单独的一个标记，还是作为另外一个标记中的一部分比如说`var a`这是个单独的标记。词法分析就是在做这件事情。

2. 解析

	把上一步做的一系列的标记解析成一个抽象语法树(简称AST)。比如说`var a = 2`，可以被解析为：

	![screenshot](http://img4.tbcdn.cn/L1/461/1/818a2c121b1fff70c3b7461ab5a1f9308d67a7ec.png)

3. 生成代码

	这一步是把AST变成可执行代码	。这里先不做详细介绍了，只知道`var a = 2`最后的结果是会创建一个变量`a`(申请一个内存空间等等的操作)，然后把2这个值存进变量`a`。

但是JS不同于传统的预编译语言如(C, Java等)，它是一门边运行边编译的语言。比如说这里的`var a = 2;`是在它被即可执行前才编译的，而不是在整个程序开始之前。这里是先编译分析`var a`的变量声明，再执行`a = 2`。

作用域(scope)介绍：

scope是指一系列规则来决定一个变量在哪里怎么被找到。这个查找过程涉及这两种LHS(setter查询，left-hand-side)、RHS(getter查询, right-hand-side）。

先解释下LHS和RHS，对于`var a = 2`：

1. 首先做RHS查询(RHS)，`var a`，查询`a`的变量，一个getter操作。
2. 然后`a = 2`，对变量a做一个赋值，属于一个setter操作。

LHS和RHS的差别在哪里？

比如这段代码:

```js
function foo(a) {
    console.log( a + b );
    b = a;
}

foo( 2 );
```

执行到`console.log( a + b )`的时候，compiler对这里做两个RHS查询，一个`a`和一个`b`。`a`在这里被赋值为2了，而`b`没有在当前作用域也就是`function foo`的`{}`里被声明。这时候，JS会抛出一个b的` ReferenceError`错误。

改一下代码(如果在浏览器环境做测试，请刷新下当前console，下面例子请都这样操作)：

```js
function foo(a) {
    b = a;
}

foo( 2 );
```

执行到`b = a;`的时候，compiler对这里的b做LHS操作，同样b也没有在`foo`中声明，可是没有报错。继续在console里输出b，会得到2的结果。

如果不在JS的严格模式下，compiler会自动对LHS查询的变量做创建工作。

在改一下代码：

```js
function foo(a) {
    console.log( a + b );
    b = a;
}

var b = 3;

foo( 2 );
```

输出结果是5；
可以看到，LHS和RHS，是会scope向上查询的，直到找到了`var b`的声明。

继续改：- -！

```js
function foo(a) {
    console.log( a + b );
    b = a;
}

foo( 2 );
var b = 3;
```

输出结果是NaN!

引出下一个话题`Hoisting`(变量提升)。

回到之前聊的JS编译过程，这段程序也是先编译`var b`, 再执行`b = 3`；整个程序编译后是这样的:

```js
function foo(a) {
    console.log( a + b ); // b 是undefined
    b = a;
}

var b;

foo( 2 );
b = 3;
```

看上面的程序，一个是声明方法，另一个是声明变量。既然编译器不是程序的书写顺序来编译，那么如果我同时有一个变量名和方法名声明都叫foo，这时候哪个优先级更高呢？

```js
foo(); // 1

var foo = function() {
    console.log( 2 );
};

function foo() {
    console.log( 1 );
}
```

可以看到变量foo是先声明的，可是方法foo是被优先编译的。编译后的代码如下:

```js
function foo() {
    console.log( 1 );
}

var foo;

foo = function() {
    console.log( 2 );
};

foo();
```

>顺便提一句，JS里的方法声明和方法变量声明是不一样的

方法变量声明：

```js
foo(); // not ReferenceError, but TypeError!

var foo = function bar() {
    // ...
};
```

这里编译为：

```js
var foo;

foo(); //不确定foo的类型是不是方法

foo = function bar() {
    // ...
};
```

方法声明：

```js
foo(); // 正常运行

function foo() {
  // ...
};
```

看下一段程序：

```js
foo(); // "b"

var a = true;
if (a) {
   function foo() { console.log( "a" ); }
}
else {
   function foo() { console.log( "b" ); }
}
```

输出结果是b! 这里涉及到两个问题：

1. JS的块作用域(`{...}`)并没有什么卵用。还是会用else里面的声明。
2. 方法声明编译的时候会被覆盖。

继续聊下一个话题---块作用域。

```js
for (var i=0; i<10; i++) {
    console.log( i );
}

i //10
```
如果JS有像C语言一样的块级作用域，那么这里的i应该会抛出`ReferenceError`的错误。可是这里输出了10，也就是说JS的块级作用域是无效的。

那么JS是怎么来实现块级作用域呢？答案是function。比如：

```js
function foo(a) {
    var b = 2;

    // some code

    function bar() {
        // ...
    }

    // more code

    var c = 3;
}
```
这里的变量`b, c`和`function bar`都是对外隐藏的，只在`foo` function的作用域里有效。

既然function可以来隔离作用域，那么我们可以轻松实现作用域隔离。

```js
var a = 2;

function foo() { // <-- insert this

    var a = 3;
    console.log( a ); // 3

} // <-- and this
foo(); // <-- and this

console.log( a ); // 2
```

但是这里又有`foo`这个函数名把全局的命名空间给污染了。JS通常的解决方式是：

```js
var a = 2;

(function foo(){ // <-- insert this

    var a = 3;
    console.log( a ); // 3

})(); // <-- and this

console.log( a ); // 2
```

这样既做到了作用域里面的变量没有污染全局，`foo`方法也没有污染全局。这种方式叫做立即执行函数表达式(IIFE--Invoking Function Expressions Immediately)

####ES6 的let与const

let与const提供了JS的块级作用域。比如回到之前的：

```js
for (let i=0; i<10; i++) {
    console.log( i );
}

console.log( i ); // ReferenceError
```

这里的`let i`就只在for循环里块级作用域中起效。

##闭包

闭包的定义：
>Closure is when a function is able to remember and access its lexical scope even when that function is executing outside its lexical scope.

简而言之，是指一个方法在它词法作用域外被调用的时候，还能引用到它词法作用域上下文的变量的方法。

举个例子：

```js
function foo() {
    var a = 2;

    function bar() {
        console.log( a );
    }

    return bar;
}

var baz = foo();

baz(); // 2 --baz() (也就是bar)在外部调用，还可以捕获到它上下文的变量`var a`
```

这里的baz就是闭包，它在外部被调用的时候还能"认识"变量`var a`，而变量`a`是在`foo`函数里声明的，并且只在`foo`函数里有效。

看一个经典的理解闭包的案例：

```js
for (var i=1; i<=5; i++) {
    setTimeout( function timer(){
        console.log( i );
    }, i*1000 );
}
```

这里的输出结果是，每一秒打印一个6，一共打印了5次。

如果是一个刚接触JS的人，可能会认为这里应该是1，2，3，4，5。每隔一秒打印一个。

首先来看，为什么是输出6。`setTimeout`函数是在for循环结束后执行的（即使`setTimeout(.., 0)`），for循环结束的时候i的值是6。所以打印了5次6.

回想之前说到的块级作用域，这里的`var i`不仅仅在for循环块级作用域中有作用，在全局作用域中也是有作用的，如果想要输出1，2，3，4，5的话，那就让循环里的每个i都只在一个单独的作用域里。之前也说到，只有function可以有独立的作用域。

所以这里尝试改成这样：

```js
for (var i=1; i<=5; i++) {
    (function(){
        setTimeout( function timer(){
            console.log( i );
        }, i*1000 );
    })();
}
```

很遗憾，结果还是一样的，打印了5个6。为什么呢？虽然这里用IIFE创建分割了循环的作用域，可是它什么都没有做，因为这是作用域啥都没干，i还是外部作用域变量的那个i，i的值没有在这个作用域中被捕获，还是循环后的6。

所以这里尝试捕获一下i的在IIFE的作用域中:

```js
for (var i=1; i<=5; i++) {
    (function(){
        var j = i;
        setTimeout( function timer(){
            console.log( j );
        }, j*1000 );
    })();
}
```

成功了！1，2，3，4，5 每隔一秒被打印了。

更优雅些：

```js
for (var i=1; i<=5; i++) {
    (function(j){
        setTimeout( function timer(){
            console.log( j );
        }, j*1000 );
    })( i );
}
```

但是说到底，我们的需求很简单，只是想要for循环的块级作用域起效果就行了，用闭包来实现作用域显得有些啰嗦。

所以回想一下之前说到的ES6的`let`：

```js
for (var i=1; i<=5; i++) {
    let j = i; // yay, block-scope for closure!
    setTimeout( function timer(){
        console.log( j );
    }, j*1000 );
}
```
这里每次，用j来创建一个有效的块级作用域就可以了。

当然ES6对for循环里的let也做了特殊的行为操作---每次循环的时候都会自动创建一个新的let i，所以可以像下面这么优雅的写了。

```js
for (let i=1; i<=5; i++) {
    setTimeout( function timer(){
        console.log( i );
    }, i*1000 );
}
```
##模块化开发

如果不是很了解前端模块化开发的价值，可以参考一下玉伯的这篇文章---[前端模块化开发的价值](https://github.com/seajs/seajs/issues/547)。

####Revealing Module

下面这段是典型的一个Revealing 模块。

```js
function CoolModule() {
    var something = "cool";
    var another = [1, 2, 3];

    function doSomething() {
        console.log( something );
    }

    function doAnother() {
        console.log( another.join( " ! " ) );
    }

    return {
        doSomething: doSomething,
        doAnother: doAnother
    };
}

var foo = CoolModule();

foo.doSomething(); // cool
foo.doAnother(); // 1 ! 2 ! 3
```

上面这段模块化的代码做到了什么：

1. 如果外部不调用`CoolModule()`方法的话，方法体里面的代码是不会被编译执行的(前面说到过JS是边编译边执行的)
2. `CoolModule`返回了一个新对象，只有新对象里的属性(闭包方法)能访问`CoolModule`这个作用域里的变量(有点像私有属性了)

####Modern Modules

现在已经有很多库在这方面都做的不错了，这里介绍个最简单的概念实现：

```js
var MyModules = (function Manager() {
    var modules = {};

    function define(name, deps, impl) {
        for (var i=0; i<deps.length; i++) {
            deps[i] = modules[deps[i]];
        }
        modules[name] = impl.apply( impl, deps );
    }

    function get(name) {
        return modules[name];
    }

    return {
        define: define,
        get: get
    };
})();
```
这段代码同时包含了模块的导出，和模块的引入功能，看下如何使用它：

```js
MyModules.define( "bar", [], function(){
    function hello(who) {
        return "Let me introduce: " + who;
    }

    return {
        hello: hello
    };
} );
```
定义bar模块，并且在这个模块里不引入其他模块。

```js
MyModules.define( "foo", ["bar"], function(bar){
    function awesome(which) {
        console.log( bar.hello( which ).toUpperCase() );
    }

    return {
        awesome: awesome
    };
} );
```
定义另外foo模块，并且在foo模块中引入bar模块。

```js
var bar = MyModules.get( "bar" );
var foo = MyModules.get( "foo" );

console.log(
    bar.hello( "hippo" )
); // Let me introduce: hippo

foo.awesome("David"); // LET ME INTRODUCE: DAVID
```

JS里的模块化，有两点必须要满足：

1. 通过一个外部的方法来创建作用域。
2. 外部方法返回的对象至少要有一个闭包来捕获方法作用域里的其他内部对象。

###一道微博流行的面试题

``` js
function Foo() {
    getName = function () { alert (1); };
    return this;
}
Foo.getName = function () { alert (2);};
Foo.prototype.getName = function () { alert (3);};
var getName = function () { alert (4);};
function getName() { alert (5);}

//请写出以下输出结果：
1. Foo.getName();
2. getName();
3. Foo().getName();
4. getName();
4. new Foo.getName();
5. new Foo().getName();
6. new new Foo().getName();
```
.

.

.

答案是：

1. 2 --- Foo对象的属性方法
2. 4 --- 编译后的代码如下

	```js
	function Foo() {
	    getName = function () { alert (1); };
	    return this;
	}

	function getName() { alert (5);}	// 函数声明高于变量声明
	var getName;

	getName = function () { alert (4);};

	Foo.getName = function () { alert (2);};
	Foo.prototype.getName = function () { alert (3);};

	getName(); // 4
	```
3. 1 --- Foo()调用过程中 1. 更改了getName变量的实现 2. Foo()返回的对象是全局对象this。所以getName()还是全局对象的调用只不过实现改了。

4、5、6首先考验的是对JS运算优先级的认识，不熟悉可以先参考[MDN的资料](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Operator_Precedence)再考虑答案是什么。

IV. 2 --- 点操作优先级高于`new`操作符，所以这里`new (Foo.getName)()`，也就是2

V. 3 --- 根据操作符优先级，首先翻译为`(new Foo()).getName()`，`new Foo()`返回一个与Foo原型链链接的新对象而不是`Foo()`返回的全局对象。这里这个新对象就去找`Foo.prototype`上的`getName`属性。

VI. 3 --- 和上一个一样，只不过是又多了个new操作。

##动态作用域

之前介绍都是词法作用域，解释了JS在编译期间是如何查找变量的规则。现在看看JS运行时候的作用域是怎么定义的，JS里的this就是动态作用域来决定的(之前一篇介绍[this](http://xiekw2010.github.io/2015/11/04/you-dont-know-jsthis-prototypes)的)。

举个例子：

```js
function foo() {
    console.log( a );
}

function bar() {
    var a = 3;
    foo();
}

var a = 2;

bar();
```

这里会输出2还是3呢？

从词法作用域角度来看，编译的时候，是先去找`foo`里的a变量，找不到，然后去找全局的a，全局的是2。所以`foo()`稳稳的输出2。

可是，在`bar`运行的时候，`foo`同样也会去找a的值，但是它首先会在`bar`的作用域里查找，所以`bar`稳稳的输出3。

所以除了考虑词法作用域(编译时查找)，动态作用域也是要考虑的(运行时查找）。
