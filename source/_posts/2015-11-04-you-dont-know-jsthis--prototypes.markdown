---
layout: post
keywords: blog
description: blog
title: "You don't know js(this & prototypes)"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-11-04 17:53:30
---

前段时间做了些H5的日常需求，详见[全球精选](https://ju.taobao.com/m/jusp/nv/haiwaiju/mtp.htm?spm=a2147.7632989.JU005.4&spm-cnt=608.6750302.0.0.G0TwsM&jhsstyle=qqjx&from=)。觉得学好JS还是很有必要的，不管它是不是世界上最好的语言。Github上看到一本书[You Dont know JS](https://github.com/getify/You-Dont-Know-JS)，这本书介绍的概念算是比较清楚的。

这周主要介绍JS的`this`。

####this
--

先来做一个题目:

```js
function foo(num) {
	console.log( "foo: " + num );
	this.count++;
}

foo.count = 0;

var i;

for (i=0; i<10; i++) {
	foo( i );
}
// foo: 0,
// foo: 1,
...
// foo: 9,

console.log( foo.count );
```

最后foo.count等于几呢 ？

答案是0。如果对this的概念不清楚的话一般会认为答案是10。

为什么呢？因为在`for`循环里调用`foo(i)`时，`this`指向的不是`foo`。如果你在游览器的环境里。可以把这段代码复制到console里测试一下。

下面介绍几个方法，可以把foo.count结果输出为10。

1. 把`this.count++`改为`foo.count++`。
2. 使用对象来包装，代码如下

	```js
	function foo(num) {
		console.log( "foo: " + num );
		this.count++;
	}

	var foo = {
		bar: foo,
		count: 0
	}

	var i;
	for (i=0; i<10; i++) {
		foo.bar(i)
	}

	```
3. 在`for`循环里把`foo(i)`改为`foo.call(foo)`或者`foo.apply(foo)`, 代码如下：

	```js

	function foo(num) {
		console.log( "foo: " + num);
		this.count++;
	}

	foo.count = 0;

	var i;
	for (i=0; i<10; i++) {
		foo.call(foo, i); // 或者foo.apply(foo, [i]);
	}


	```

4. 使用bind，代码如下：

	```js
	function foo(num) {
		console.log( "foo: " + num );
		this.count++;
	}

	foo.count = 0;

	var i;
	var bar = foo.bind(foo);

	for (i=0; i<10; i++) {
		bar( i );
	}
	```


为什么在`for`循环里调用`foo(i)`时，`this`指向的不是`foo`，它指向的是什么呢？

现在可以重新打开一下游览器的console，再输入最开始的那段代码。接着在console里输入`foo.count`，结果应该是0，再继续看下去。

然后`console`里输入`window.count`，结果是NaN。那么什么时候改了window.count的属性呢？(window没有count属性，可以重新打开游览器console，输入windown.count，结果应该是undefined).

就是在`for`循环里调用`foo(i)`的时候。

再来看一下解决方案2，用对象包装的方式，`foo.bar(i)`调用，最后调用者`foo`的`count`属性被改变了。所以`this`的指向决定于调用方法的调用地点(后面用__call-site__来指代这个)。

理解**call-site**先来看个例子：

```js
function baz() {
    // call-stack is: `baz`
    // so, our call-site is in the global scope

    console.log( "baz" );
    bar(); // <-- call-site for `bar`
}

function bar() {
    // call-stack is: `baz` -> `bar`
    // so, our call-site is in `baz`

    console.log( "bar" );
    foo(); // <-- call-site for `foo`
}

function foo() {
    // call-stack is: `baz` -> `bar` -> `foo`
    // so, our call-site is in `bar`

    console.log( "foo" );
}

baz(); // <-- baz的call-site是global对象，在游览器里也就是window。
```

所以理解了这个，再做个实验。还是将第一段代码copy进console，但是先等等，先在console输入`window.count=0`。然后再把第一段把copy进console。再输入`window.count`，可以看到结果是10。所以`foo`方法里的`this`指向的是call-site对象也就是js context的global对象`window`。

但是，除了**call-site**外，我们看到前面的解决方法3和4。`bind`, `apply`, `call`。那么这些规则和**call-site**的优先级呢？除了这些规则还有什么规则。下面就人肉列举一下：

#####默认绑定
--

默认绑定就是**call-site**绑定，用的时候需要注意一下js的'use strict'。

```js
function foo() {
	"use strict";

	console.log( this.a );
}

var a = 2;

foo(); // TypeError: `this` is `undefined`
```
因为a是全局对象，foo()的**call-site**(window)调用a的时候，window对象在strict模式下不存在。所以a找不到，不然就会打印出2.


#####隐式绑定
--

还是由**call-site**的位置决定，如下例：

```js
function foo() {
	console.log( this.a );
}

var obj2 = {
	a: 42,
	foo: foo
};

var obj1 = {
	a: 2,
	obj2: obj2
};

obj1.obj2.foo(); // 42
```

有风险的是，很容易会造成隐式绑定丢失的问题，如下例：

```js
function foo() {
	console.log( this.a );
}

var obj = {
	a: 2,
	foo: foo
};

var bar = obj.foo; // function reference/alias!

bar(); // undefined
obj.foo(); // 2
```
虽然bar看上去是obj.foo的引用，但其实它最后指向的是foo, foo虽然是obj的对象，但不属于obj。

所以这里就走到了**默认绑定**规则上去了。


#####显式绑定

就是之前提到的方案3和4，这里顺便介绍一下这三个方法：

```js
	function foo(arg1, arg2, arg3) {
		console.log(this);
	}
```

1. call 用法: `foo.call(obj, arg1, arg2, arg3)`
2. apply 用法: `foo.call(obj, [arg1, arg2, arg3])`

	两个方法都显示指定了foo方法里this所指的对象是obj。不同的是call对foo方法的参数是一个一个的传，而apply是把他们放到一个数组里一下子传进去。apply最后参数拿出来也是按传的顺序拿出来的。

3. bind 用法: `var bar = foo.bind(obj, arg1, arg2, arg3);`

	它返回一个和foo实现一样的新函数，其他和call一样。

#####New 绑定

```js
something = new MyClass(..);
```

抛开传统的面向对象概念，这段代码做了以下事情：

1. 一个全新的对象被创建了。
2. 新对象的this就是MyClass方法里的this。
3. 除非MyClass里返回的是其他对象，不然就是this对象。

如下：

```js
function foo(a) {
	this.a = a;
}

var bar = new foo( 2 );
console.log( bar.a ); // 2
```

#####顺序

按照规则来，绑定的优先级顺序是**new binding** > **explicit binding** > **implicit binding** > **default binding**

#####间接寻址

```js
function foo() {
	console.log( this.a );
}

var a = 2;
var o = { a: 3, foo: foo };
var p = { a: 4 };

o.foo(); // 3
(p.foo = o.foo)(); // 2
```

这里`(p.foo = o.foo)`返回的是`foo`对象而不是`p.foo`所以log了全局的a对象。
写成这样就没有问题了。

```js
p.foo = o.foo;
p.foo(); // 4
```

#####ES6 箭头函数的this

```js
function foo() {
	// return an arrow function
	return (a) => {
		// `this` here is lexically adopted from `foo()`
		console.log( this.a );
	};
}

var obj1 = {
	a: 2
};

var obj2 = {
	a: 3
};

var bar = foo.call( obj1 );
bar.call( obj2 ); // 2, not 3!
```
为什么是this绑定的是obj1的对象而不是obj2的对象呢？

`foo.call(obj1)`的时候已经绑定了`obj1`是它的this对象，也就是`bar`的this对象是`obj1`, 箭头函数在其作用域内一旦被绑定this，那它就永远不会被修改，即使是new操作符。

###总结

虽然this的规则挺多挺绕的，但是优雅的js代码离不开它。
