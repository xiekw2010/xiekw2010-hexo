---
layout: post
keywords: blog
description: blog
title: "You don't know js(prototypes)"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-12-28 17:53:30
---

`__proto__` vs `prototype`

先从一个实验开始了解一下这两个名词。（***注意***: `__proto__`在有些场合下可能被写为`[[prototype]]`, 这里以chrome console里的写法为准）

把这段代码复制到chrome console中(下面简称console)：

```js
/* 代码一 */

var a = {};
a.prototype; // undefined;
a.__proto__; // Object {};

function Foo() {}
Foo.prototype; // Foo{};
Foo.__proto__; // Function(){};

var b = new Foo();
b.prototype; // undefined;
b.__proto__; // Foo{};
```

可以看到普通对象没有`prototype`属性，而function对象有。并且通过new操作符 + function对象调用得到对象的`__proto__`指向function对象的`prototype`。这里都是`Foo{}`。

继续：

```js
/* 代码二 */

b.name; // undefined;
Foo.prototype.name = 'name';
b.name; // 'name';
```

在js里，每个对象都有一个`__proto__`属性，指向另外一个对象的`prototype`，所有对象的root 对象是`Object{}`。

```js
/* 代码三 */

a.__proto__; // Object {};
Object.prototype; // Object {};
Foo.prototype.__proto__ ; // Object {};
```

一个对象在`[[getter]]`某个属性值的时候会现在自身对象上找，如果找不到，会通过它的隐藏属性`__proto__`上去找，如果再找不到，继续去它的`__proto__.__proto__`上去找，直到找到`root.prototype` 对象。在__代码一__中可以继续做测试：

```js
/* 代码四 */

Object.prototype.FindMe = 'FindMe';
a.FindMe;  //'FindMe'
Foo.FindMe; //'FindMe'
b.FindMe; //'FindMe'
```

##constructor

先来搞清楚一个事实，__代码一__中的a对象，默认有一个`a.constructor`属性，这个属性指向的是`a.__proto__.constructor`也就是`Foo.prototype.constructor`对象。

```js
/* 代码五 */

function Bar() {}
Bar.prototype.constructor = 'somebar';
var abar = new Bar();
abar.constructor; // 'somebar'
```

那么`prototype.constructor`的`constructor`对象什么时候用到呢？--在与new 操作符打交道的时候用到，new 操作符可以这么理解：

```js
/* new 构造器 */

Function.prototype.method = function (name, func) {
  if (!this.prototype[name]) {
    this.prototype[name] = func;
  }

  return this;
}

// 如果new运算符是一个方法而不是运算符
// 这里的this可以理解为prototype.constructor
Function.method('new', function () {
// 创建一个新对象， 它链接于构造器函数的prototype
  var that = Object.create(this.prototype);

// 调用构造器函数，绑定 this 到新对象
  var other = this.apply(that, arguments);

// 如果它的返回值不是一个对象，就返回该新对象
  return (typeof other === 'object' && other) || that;
})
```

##shadowing

__代码五__里`abar.constructor`指向`Foo.prototype.constructor`。这里做`[[getter]]`操作读取属性没有问题。如果做`[[setter]]`操作会发生什么呢？(这里撇开`constructor`本身的意义，只是介绍setter on `__proto__`);

```js
/* 代码六 */

abar.constructor; // 'somebar'
abar.constructor = 'aBar';
abar.constructor; // 'aBar';
Bar.prototype.constructor; // 'someBar';
abar.__proto__.constructor; // 'someBar';
```

如果一个对象的属性，这里是`constructor`同时存在于它自身或者它的`__proto__` chain上，那么就称这个情况为shadowing。下面讨论`myObject.foo = "bar"`三种情况下的表现，当foo不在myObject上，但在myObject的`__proto__`上。

工具方法`Object.getOwnPropertyDescriptor(Bar, 'constructor')`, 得到对象上某个属性的描述。

1. 如果`foo`在`__proto__` chain上找到，并且***它不是read-only(`writeable: false)***属性，那么就产生正常的shadowing属性情况。
2. 如果`foo`在`__proto__` chain上找到，但是***它是read-only(`writeable: true)***属性，那么在foo上set这个属性或者新建这个属性是不允许的。
3. 如果`foo`在`__proto__` chain上找到，并且在定义这个属性的时候设置了它的`[[setter]]`，那么`foo`不会被加到`myObjet`上，`foo`的setter也不会被重新定义。

在#2和#3这种情况下，如果要重新定义`foo`属性，那就需要使用`Object.defineProperty(..)`方法了。

##class

######实例化 & 类继承？

传统面向对象语言(以下简称OO)有class的概念。class有两个重要特性--实例化和类继承。

实例化：根据class的模型copy出一个具有该模型特征的新对象。

类继承：子类可以继承或者父类的属性，方法。

那么js是怎么做到这两项的呢？

之前提到了`new Foo()`这种像极了传统OO方式的新建对象操作。那么`var a = new Foo();`到底做了哪些具体的事情呢？

1. 一个全新的对象被创建了。
2. 这个新对象的`__proto__`被link了。link到`Foo.prototype`上。
3. 新对象的被当做this出现在Foo()调用的上下文中。
4. 除非Foo()返回一个其他的对象，不然就自动返回这个新建的对象。

这里主要看第二步，新对象的`__proto__`被link了，这意味着这个新对象以后所有的属性查找，找不到就自动会delegate到`Foo.prototype`上去。只要对`Foo.prototype`做添加属性，也就对新对象做了添加属性。copy操作就这么被实现了，很像传统OO的实例化操作了。只不过是从对象link到对象，没有class的参与。

上面看到的操作是function 与 object之间的通过new link。那么普通的object 与 object之间如何link呢？还记得代码***new 构造器***里提到的`Object.create(...)`吗？`Object.create(...)`会新建一个对象，并把这个对象的`__proto__`link到参数里对象的`__proto__`上。

那么怎么做到js的‘class’继承呢(如下图)？

![fig3](http://img2.tbcdn.cn/L1/461/1/6763693783caf43da914d21c3de411357f85218f.png)

很简单一句代码`Bar.prototype = Object.create(Foo.prototype)`;

把两个原型链link，那么b1 b2会自动有a1 a2的属性和方法。具体实现：

```js
function T1() {};
function T2() {};

T1.prototype.laoluo = 'laoluo';
T2.prototype = Object.create(T1.prototype);

var t1 = new T1();
t1.laoluo; //'laoluo';

var t2 = new T2();
t2.laoluo; //'laoluo';
```

关于`Object.create(...)`通常两种误解，认为下面两种实现效果是一样的：

	1. T2.prototype = T1.prototype;
	2. T2.prototype = new T1();

第一种直接直接把`T2.prototype`指向了`T1.prototype`。对`T2.prototype.change`操作也就是对`T1.prototype.change`做操作。这两个是一个对象了，并不是想要的***新建***一个对象然后把两个对象link到一起。

第二种，可以说和`Object.create(...)`效果是一样的。但问题在于`T1()`，T1方法调用的时候，天知道里面做了什么事情（比如log，改变某些全局对象state等）。这里是要Link 两个对象的`__proto__`，用new的话，没有`Object.create()`来的纯粹。

```js
function T1() {
  console.log('creating T1 now...');
};

T2.prototype = new T1(); // `creating T1 now...` 可能调用者不需要这段log
```

##OO style 与 OLOO style的比较

OO：object-oriented;
OLOO: objects-linked-to-other-objects;

在js里，有时候用oo思想来实现一些东西，远没有用OLOO思想来的clean。

OO style:

```js
function Foo(who) {
	this.me = who;
}
Foo.prototype.identify = function() {
	return "I am " + this.me;
};

function Bar(who) {
	Foo.call( this, who );
}
Bar.prototype = Object.create( Foo.prototype );

Bar.prototype.speak = function() {
	alert( "Hello, " + this.identify() + "." );
};

var b1 = new Bar( "b1" );
var b2 = new Bar( "b2" );

b1.speak();
b2.speak();
```
OLOO style:

```js
Foo = {
	init: function(who) {
		this.me = who;
	},
	identify: function() {
		return "I am " + this.me;
	}
};

Bar = Object.create( Foo );

Bar.speak = function() {
	alert( "Hello, " + this.identify() + "." );
};

var b1 = Object.create( Bar );
b1.init( "b1" );
var b2 = Object.create( Bar );
b2.init( "b2" );

b1.speak();
b2.speak();
```

可以看到用OLOO style就不需要用js所不擅长的new和prototype来实现了class了，易读性更强一些。

##总结

js没有class只有object（ES6的class也只是语法糖），从一个`object.__proto__` link 到另一个`object.__proto__`来达到copy。link的方式有两种:

	1. var anotherobj = new Obj();
	2. var anotherobj = object.create(obj);

>js是一门弱类型语言，从不需要类型转换。对象继承变得无关紧要。对一个对象来说重要的是它能做什么，而不是它从哪里来。 ---《Javascript语言精粹》


P.S 这篇文章是我写的第二遍，第一遍清除mac垃圾的时候被我误删了，本来要发表了，结果找不到了，哭晕在厕所。不过第二遍我觉得思路清晰了好多，质量也高了好多。
