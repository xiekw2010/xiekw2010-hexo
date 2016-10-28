---
layout: post
keywords: blog
description: blog
title: "一次RN跨平台开发之旅GitFeed"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2016-02-11 17:53:30
---

前段时间因为工作需要接触了一下react-native(下面简称RN)。

对于一个传统的客户端开发码农来说，RN重新让我认识了客户端开发。

请随便使用什么客户端扫一扫。

iOS版：

![2016_02_11_0840321202](http://img3.tbcdn.cn/L1/461/1/b77fe7c74ef98edde9ff366416fb5597bd8eb88c.png)

android版：

![2016_02_11_0841031457](http://img3.tbcdn.cn/L1/461/1/d395f6d2148b392550b971ce7f50672b04fd1716.png)

下面聊一聊一些开发过程中令我比较爽的点吧。

##90% write once，run both(iOS & android)

网络：

  RN底层的网络使用的是JS虚拟机自带的`XMLHttpRequest`，对外统一封装了一层fetch的JS接口，保证了两个平台的一致性。

存储：

  同样，两个平台实现了各自的持久化类(iOS里是NSCoding的方式)，再bridge到统一的JS接口。

UI视图：

  RN对两个平台也做了最基本的View，Image，Text，ScrollView，ListView，Gesture等接口相同的JS封装。

对于Image：

  * 保证downloading，decoding是在native的背景线程操作的。
  * assets的管理两个平台可以都通过JS require入native的资源文件。而不是通过各自平台的图片资源导入方式。

对于Text：

  保证measure是在native的背景线程操作的。

对于ListView：

  这也是RN被很多人诟病的一点，当ListView展示一个很长并且图很多的列表时，内存占用过高，它的实现没有用到native的recycle机制。

  为什么ListView RN不提供两个平台各自的recycle机制的ListView呢？FB的vjeux做了如下[解释](https://github.com/facebook/react-native/issues/499)

  简单介绍下有以下几点(观点会比较基于iOS的实现)：

  1. 加载均衡

    因为recycleView里的某个元素出现在屏幕上时，它是需要***同步***渲染的，这个操作最好在16.7ms内完成。但是用RN的JS <=> Native通信的异步机制做到这点比较难。如果这里用同步机制来实现，如果新元素结构复杂一点，那也很难保证JS <=> Native的通信时间加上Native的渲染时间在16.7ms内完成而不掉帧。

    ListView做到的优化是：当滚动到距离屏幕底一定距离时，预渲染下一屏的row。并且通过`requestAnimationFrame`来保证每一个row都是在一帧内完成。

  2. 重用机制的不友好

    iOS的tableView重用机制在当cell比较复杂的时候对于开发人员比较难实现，并且也很容易出现UI bug。每个cell要有自己的内部状态(比如视屏的播放状态，字的输入状态，水平ScrollView滚动状态等...)， 当这种cell需要被重用的cell恢复状态时，必须把他们当时的状态重新恢复，这里牵涉外部需要对每一个cell内部状态的管理。

  3. 内存管理

    对于内存，FB做的优化是，对于离开屏幕的元素，会把他们从`dom tree`里删除掉，但是不把他们的`virtual dom`引用删除掉，以保证下次这些元素重新出现在屏幕上时，他们的内部状态还是当时的。

  4. 改变只改变的。

    每一个ListView都有一个dataSource来对应其内部的元素数据。与FB目前的这种实现契合的优化是，假如前一个dataSource有1000个元素，后一个dataSource也有1000个元素，ListView对这两个dataSource做一个diff，然后只改变改变的data对应的row元素。

  5. 不同高度的布局

    iOS的TableView，当要渲染不同高度的cell的时候，必须在cell被渲染前提前计算出。(有些计算还比较蛋疼，比如说字的高度，因为需要同步进行，可能会阻塞主线程，多扯几句，目前比较好的框架级解决方案可以看看[AsyncDisplayKit](http://asyncdisplaykit.org/)的异步计算与渲染)。

    因为RN有自己的异步布局系统，所以用ListView就可以避免那些蛋疼的手动布局计算。一个row长什么样，有多高，渲染的时候一次性搞定。

UI动画：

  目前RN有两套动画系统---`Animated`和`LayoutAnimation`。`Animated`更注重小而精细的动画控制，`LayoutAnimation`更关注全局布局类型的动画。

  Animated系统没有bridge到native的动画系统。是自己用JS实现了一套动画系统(JS timer & nativeProps)，为了平台兼容性。

  总结: 整个过程如图所示

  ![3](http://img3.tbcdn.cn/L1/461/1/22fdbf5e9a519fcfcdbcd715d429fffd221d911e.png)

Navigator：

FB提供一个iOS和android可以通用的导航器。比起传统的UINavigationViewControler，activity的`pushViewController`和`startActivity`，FB的`Navigator`是基于URL的导航，更加贴切于方便组件化的开发。

平台UI展示差异：

这里说90%是可以重用的代码。那剩下的10%是什么呢？

比如iOS的native设计是注重扁平和模糊透明，而Android的native设计是matrial design；比如Android有toast组件，而iOS平台自身没有这个组件，如果一个JS api要两个平台同时跑的话，那得自己实现一个iOS的toast，然后按照RN方便的桥接方式Bridge到JS；比如iOS官方App喜欢底部tabbar导航，而安卓官方喜欢左侧的抽屉式导航(个人还是想吐槽一下这个低效的设计)...

##所改即所见

传统的客户端开发，每次改动都要重新编译和构建。即使是调一个UI视图的几个像素，也要等那么久。而不像web开发那样，保存一下文件，刷新一下游览器就可以了。

而到了RN，强大的调试工具使得开发效率大幅提升。还是同样调整几个像素，我也只要保存一下JS文件然后CMD+R一下模拟器就能马上看到我改动的东西了。no more compile and build again!

chrome 也提供了很方便的JS代码debugger。

##描述性布局

对于一个动态变化的界面，往往会有add，remove，move等对subview的操作，这些操作可能散布在某个文件各个片段或者某些文件的各个片段。这些代码如果没有好好管理好，或者约定好，对于开发人员来说，尤其对于接手的新成员来说，并不是那么一目了然，业务复杂点奇葩点，那么维护起来是有些蛋疼的。

而到了RN，每个组件只有一个方法(`render`)用来描述这个组件长什么样。没有了对视图的add，remove，move的操作。每个组件在`render`里根据自身的不同内部状态来决定自己该有什么组件组成。 并且FB通过JSX语法给开发人员了一种描述性布局体验。

通过'内联css'来做到对UI的啰嗦代码进行分离。布局方案采用css3的flexbox，无论在web，还是在iOS和android，都是learn once，write anywhere。

##Re-render everything

所有与view绑定的数据变化时，不用再一个个去找这些数据将要影响的那些view的布局或者展示，只需要将当前组件重新刷新一下就行了。react强大的virtual dom diff算法会帮我搞定哪些子view需要重新刷新，哪些又不需要。

##快速发布(codePush)

对于发布和托管app的js bundle，我选择了巨硬的[code-push](http://microsoft.github.io/code-push/)。有几个点还不错：

1. 免费的代码存储服务，cdn下载服务.（至少现在beta版是免费的）
2. 配套的统一的JS接口，iOS和android各自native端下载更新的实现。完善的CLI工具。
3. 提供`A/B test`的发布功能。

##有些不足

1. 版本更新不兼容。
2. ListView 多图长列表内存消耗还是大。

##RN技术栈与工具

- flexbox布局
- ES6 & ES5, 尤其是promise
- react 了解

##总结
快速开发，快速发布，又可以兼容两个平台。native开发真的要失业了吗？

##参考资料

- [react-native-bringing-modern-web-techniques-to-mobile](https://code.facebook.com/posts/1014532261909640/react-native-bringing-modern-web-techniques-to-mobile/)--[中文版](http://gold.xitu.io/entry/5693765a60b27e9ba8e055cf)
- [code-push](http://microsoft.github.io/code-push/)
- [ListView performance](https://github.com/facebook/react-native/issues/499)
- [React Architecture](https://code.facebook.com/videos/1507312032855537/oscon-2014-react-s-architecture/)
