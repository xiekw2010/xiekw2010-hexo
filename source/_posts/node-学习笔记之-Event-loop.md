---
title: node 学习笔记之 Event loop
date: 2017-04-09 17:46:22
tags:
---

每当聊起 Event loop 时，我们自然会想到，这不就是一个while(true)的事件循环嘛，当任务队列里有任务时就执行。

那么这个while(true)的原理是什么呢？

## 什么是 Event loop ?

WIKI 定义：

>In computer science, the event loop, message dispatcher, message loop, message pump, or run loop is a programming construct that waits for and dispatches events or messages in a program.

Event loop 是一种程序结构，是实现异步的一种机制。Event loop 可以简单理解为：

1. 所有任务都在主线程上执行，形成一个执行栈（execution context stack）。

2. 主线程之外，还存在一个"任务队列"（task queue）。系统把异步任务放到"任务队列"之中，然后主线程继续执行后续的任务。

3. 一旦"执行栈"中的所有任务执行完毕，系统就会读取"任务队列"。如果这个时候，异步任务已经结束了等待状态，就会从"任务队列"进入执行栈，恢复执行。

4. 主线程不断重复上面的第三步。

乍看这几个步骤可能有点抽象，看下这张 gif 图就能明白了：

![](https://img.alicdn.com/tfs/TB1gaAkQpXXXXXMXVXXXXXXXXXX-657-376.gif)

对 JavaScript 而言，Javascript 引擎／虚拟机（如 V8 ）的执行 stack 是单线程的，既上图中的 stack 就是 js 虚拟机，并且假设是运行在游览器的环境中。

参照上图：

在 js stack 上依次执行了 console.log, $.get, console.log 了三个函数，然后 stack 上一个空的状态。

其中 $.get 把一个 callback 通过 webapi 放到了 task queue 中，这个 task queue 又通过 event loop 把这个 callback function 放到了 stack 上执行。

可以看到，js 虚拟机就是单线程执行栈，运行环境（浏览器，node）负责把异步任务放入到任务队列中，当执行引擎的线程执行完毕（空闲）时，运行环境就会把任务队列里的（执行完的）任务（的数据和回调函数）交给引擎继续执行，这个过程是一个***不断循环***的过程，称为***事件循环***。

***注意：JavaScript（引擎）是单线程的，Event loop 并不属于 JavaScript 本身，但 JavaScript 的运行环境是多线程／多进程的，运行环境实现了 Event loop。***

## node 的 Event loop

上文中说到，Event loop 就是通过运行环境（node，游览器）调用js api，把传入的 callback 放到 js call stack 上的运行一种机制。

如果是最简单的一种机制，那么就是挨个依次执行这个队列里的每个任务。

但 node 不是最简单的，它的执行机制如下所示：

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

> 图中的每个盒子可以理解 event loop 执行的各个『阶段』，每个『阶段』维护着各自类型的 callback 队列，当一个『阶段』callback 队列中没有任务或者超过已经满额执行完队形任务后，当前这个 event loop 会进入到下一个『阶段』，从一个『阶段』到下一个『阶段』，以此往复。

### 阶段类型总览：

- timers: 这个阶段执行 setTimeout() 和 setInterval() 设定的回调。
- I/O callbacks: 执行几乎所有『异常关闭』的回调，以及 setImmediate()
- idle, prepare: 仅内部使用。
- poll: 获取新的 I/O 事件；node 会在适当条件下阻塞在这里。
- check: 执行 setImmediate() 设定的回调。
- close callbacks: 执行比如 socket.on('close', ...) 的回调。

### Phases in Detail 阶段详情

#### timers

一个 timer 其实只是指定了它的 callback 尽量尽早的在指定的那个时间后（这个时间通常称为下限时间）执行，比如指定 100ms，但是事实往往是 > 100ms 后执行，因为『操作系统调度』或者其他运行的 callback 太耗时而延迟执行了这个 callback。

#### I/O callbacks

这个阶段执行一些系统操作的回调。比如 TCP 错误，如一个 TCP socket 在想要连接时收到ECONNREFUSED,
类 unix 系统会等待以报告错误，这就会放到 I/O callbacks 阶段的队列执行。

#### poll

poll 阶段有两个主要功能：

- 执行下限时间已经达到的 timers 的回调，然后处理 poll 队列里的事件。
- 处理 poll 队列里的事件。

当 event loop进入 poll 阶段，并且 没有设定的timers（there are no timers scheduled），会发生下面两件事之一：

- 如果 poll 队列不空，event loop会遍历队列并同步执行回调，直到队列清空或执行的回调数到达系统上限；

- 如果 poll 队列为空，则发生以下两件事之一：

	- 如果代码已经被setImmediate()设定了回调, event loop 将结束 poll 阶段进入 check 阶段来执行 check 队列（里的回调）。
	- 如果代码没有被 setImmediate() 设定回调，event loop 将阻塞在该阶段等待回调被加入 poll 队列，并立即执行。

	
但是，当 event loop 进入 poll 阶段，并且 有设定的 timers，一旦 poll 队列为空（poll 阶段空闲状态）：

- event loop 将检查 timers,如果有 1 个或多个 timers 的下限时间已经到达，event loop将绕回 **timers** 阶段，并执行 **timer** 队列。

#### check

这个阶段允许在 poll 阶段结束后立即执行回调。如果 poll 阶段空闲，并且有被 setImmediate() 设定的回调，event loop会转到 check 阶段而不是继续等待。

setImmediate()实际上是一个特殊的timer，跑在event loop中一个独立的阶段。它使用libuv的API
来设定在 poll 阶段结束后立即执行回调。

通常上来讲，随着代码执行，event loop终将进入 poll 阶段，在这个阶段等待 incoming connection, request 等等。但是，只要有被setImmediate()设定了回调，一旦 poll 阶段空闲，那么程序将结束 poll 阶段并进入 check 阶段，而不是继续等待 poll 事件们 （poll events）。

#### close callbacks

如果一个 socket 或 handle 被突然关掉（比如 socket.destroy()），close事件将在这个阶段被触发，否则将通过process.nextTick()触发。

## 概念比较

#### setImmediate() vs setTimeout()

setImmediate() 和 setTimeout()是相似的，区别在于什么时候执行回调：

1. setImmediate() 被设计在 poll 阶段结束后立即执行回调。
2. setTimeout() 被设计在指定下限时间到达后执行回调。

```js
// timeout_vs_immediate.js
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```

结果是：

```js
$ node timeout_vs_immediate.js
immediate
timeout
```

在 IO cycle 的回调中，setImmediate 永远先执行，因为 readFile 的 callback 是在 poll 阶段执行的，在 poll 的下一个阶段 check 阶段中执行的。

### 理解 process.nextTick()

直到现在，我们才开始解释process.nextTick()。因为从技术上来说，它并不是event loop的一部分。相反的，process.nextTick()会把回调塞入nextTickQueue，nextTickQueue将在当前操作完成后处理，不管目前处于event loop的哪个阶段。

看看我们最初给的示意图，process.nextTick()不管在任何时候调用，都会在所处的这个阶段最后，在event loop进入下个阶段前，处理完所有nextTickQueue里的回调。

#### process.nextTick() vs setImmediate()

两者看起来也类似，区别如下：

process.nextTick() 立即在本阶段执行回调；
setImmediate() 只能在 check 阶段执行回调。

## TLDR;

- Event loop 是一种异步方式的实现，需要单线程的执行栈，运行环境异步 api，最后才是 while(true) 运行环境异步 api 维护的 taskQueue，这一整套体系才是 Event loop
- node 的 Event loop 的 taskQueue 分布在多个阶段中，这多个阶段按一个的顺序在执行。

## 参考

- [官网文档 event-loop-timers-and-nexttick](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll)
- [翻译文档](https://github.com/creeperyang/blog/issues/26)