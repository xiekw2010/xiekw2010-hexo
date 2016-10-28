---
layout: post
keywords: blog
description: blog
title: "Advanced LLDB tech"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-02-26 17:53:30
---

今天看了wwdc2013年的advanced LLDB介绍，知道了些原来不知道的。建议直接看[objcio](http://www.objc.io/issue-19/lldb-debugging.html)里的介绍，质量很高！

#####先来一些常用的命令, 摘自[here](http://lldb.llvm.org/lldb-gdb.html):
1. p vs po vs expr
2. fr v (打印所有本地变量)
3. ta v (打印所有global变量)
4. bt (打印上下文信息)
5. wa s v someVar (set watchpoint)
6. w l (show watchpoint list)
7. wa del x (delete x watchpoint)

######理解:

1. 总结: 三者都可以执行方法(如po [foo bar]);po可以打印任何东西而p和expr只能打印基础类型，对象类型的东西只能打印出内存地址;这三者每次调用, 如果后面跟着的是对象不是方法，那么每个被调用的对象会被记录在lldb里，作为$0,$1,$2…(有个小技巧，检查某个对象是否被释放掉可以用po $0.description来查看)
	+ 推荐: 都用po来做事情，p和expr一个意思。在命令行里objc尽量使用message send的方式，避免用点操作(po [[[self navigationController] navigationBar] setBarTintColor:[UIColor redColor]])


#####也许有用的:

1. Add symoblic breakpoint
2. breakpoint add action
3. breakpoint add condition
4. watch variable change

#######理解

1. 只能在现有类的实现方法里添加，如(-[BlurEffectViewController viewWillDisappear:])
2. 当断点执行到某一行后，可以添加action来做额外的事情。可以有的想象是，加一个修改个啥对象的属性，但不用在源代码里添加这个测试修改，再继续进程。如(‘po [[[self navigationController] navigationBar] setBarTintColor:[UIColor purpleColor]]’ + ’c‘)
3. 只满足了一定条件才执行这个break point。可以有的想象是，当某个方法的某个参数，或者某个变量是某个特定的值的时候，我这断点才执行进去。
4. 可以观察某个变量的变化，当某个变量发生变化时，会自动跳到发生变化的代码上去。
