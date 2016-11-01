---
layout: post
keywords: blog
description: blog
title: "一次RN跨平台开发之旅 GitFeed"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2016-02-11 17:53:30
---

前段时间因为工作需要接触了一下 react-native (下面简称 RN )。

对于一个传统的客户端开发码农来说，RN 重新让我认识了客户端开发。

请随便使用什么客户端扫一扫。

iOS 版：

![2016_02_11_0840321202](http://img3.tbcdn.cn/L1/461/1/b77fe7c74ef98edde9ff366416fb5597bd8eb88c.png)

android 版：

![2016_02_11_0841031457](http://img3.tbcdn.cn/L1/461/1/d395f6d2148b392550b971ce7f50672b04fd1716.png)

下面聊一聊一些开发过程中令我比较爽的点吧。

## 90% write once，run both(iOS & android)

网络：

  RN底层的网络使用的是 JS 虚拟机自带的`XMLHttpRequest`，对外统一封装了一层 fetch 的 JS 接口，保证了两个平台的一致性。

存储：

  同样，两个平台实现了各自的持久化类(iOS 里是 NSCoding 的方式)，再 bridge 到统一的 JS 接口。

UI 视图：

  RN 对两个平台也做了最基本的 View，Image，Text，ScrollView，ListView，Gesture 等接口相同的JS封装。

对于 Image：

  * 保证 downloading，decoding 是在 native 的背景线程操作的。
  * assets 的管理两个平台可以都通过 JS require 入 native 的资源文件。而不是通过各自平台的图片资源导入方式。

对于 Text：

  保证 measure 是在 native 的背景线程操作的。

对于 ListView：

  这也是 RN 被很多人诟病的一点，当 ListView 展示一个很长并且图很多的列表时，内存占用过高，它的实现没有用到 native 的 recycle 机制。

  为什么 ListView RN 不提供两个平台各自的 recycle 机制的 ListView 呢？FB 的 vjeux 做了如下[解释](https://github.com/facebook/react-native/issues/499)

  简单介绍下有以下几点(观点会比较基于 iOS 的实现)：

  1. 加载均衡

    因为 recycleView 里的某个元素出现在屏幕上时，它是需要***同步***渲染的，这个操作最好在 16.7ms 内完成。但是用 RN 的 JS <=> Native 通信的异步机制做到这点比较难。如果这里用同步机制来实现，如果新元素结构复杂一点，那也很难保证 JS <=> Native 的通信时间加上 Native 的渲染时间在 16.7ms 内完成而不掉帧。

    ListView 做到的优化是：当滚动到距离屏幕底一定距离时，预渲染下一屏的 row。并且通过`requestAnimationFrame`来保证每一个 row 都是在一帧内完成。

  2. 重用机制的不友好

    iOS 的 tableView 重用机制在当 cell 比较复杂的时候对于开发人员比较难实现，并且也很容易出现 UI bug。每个 cell 要有自己的内部状态(比如视屏的播放状态，字的输入状态，水平 ScrollView 滚动状态等...)， 当这种 cell 需要被重用的 cell 恢复状态时，必须把他们当时的状态重新恢复，这里牵涉外部需要对每一个 cell 内部状态的管理。

  3. 内存管理

    对于内存，FB 做的优化是，对于离开屏幕的元素，会把他们从`dom tree`里删除掉，但是不把他们的`virtual dom`引用删除掉，以保证下次这些元素重新出现在屏幕上时，他们的内部状态还是当时的。

  4. 改变只改变的。

    每一个 ListView 都有一个 dataSource 来对应其内部的元素数据。与 FB 目前的这种实现契合的优化是，假如前一个 dataSource 有 1000 个元素，后一个 dataSource 也有 1000 个元素，ListView 对这两个 dataSource 做一个 diff，然后只改变改变的 data 对应的 row 元素。

  5. 不同高度的布局

    iOS 的 TableView，当要渲染不同高度的 cell 的时候，必须在 cell 被渲染前提前计算出。(有些计算还比较蛋疼，比如说字的高度，因为需要同步进行，可能会阻塞主线程，多扯几句，目前比较好的框架级解决方案可以看看[AsyncDisplayKit](http://asyncdisplaykit.org/)的异步计算与渲染)。

    因为 RN 有自己的异步布局系统，所以用 ListView 就可以避免那些蛋疼的手动布局计算。一个 row 长什么样，有多高，渲染的时候一次性搞定。

UI 动画：

  目前 RN 有两套动画系统---`Animated`和`LayoutAnimation`。`Animated`更注重小而精细的动画控制，`LayoutAnimation`更关注全局布局类型的动画。

  Animated 系统没有 bridge 到 native 的动画系统。是自己用 JS 实现了一套动画系统(JS timer & nativeProps)，为了平台兼容性。

  总结: 整个过程如图所示

  ![3](http://img3.tbcdn.cn/L1/461/1/22fdbf5e9a519fcfcdbcd715d429fffd221d911e.png)

Navigator：

FB 提供一个 iOS 和 android 可以通用的导航器。比起传统的 UINavigationViewControler，activity 的`pushViewController`和`startActivity`，FB 的`Navigator`是基于 URL 的导航，更加贴切于方便组件化的开发。

平台 UI 展示差异：

这里说 90% 是可以重用的代码。那剩下的 10% 是什么呢？

比如 iOS 的 native 设计是注重扁平和模糊透明，而 Android 的 native 设计是 matrial design；比如 Android 有 toast 组件，而 iOS 平台自身没有这个组件，如果一个 JS api 要两个平台同时跑的话，那得自己实现一个 iOS 的 toast，然后按照 RN 方便的桥接方式 Bridge 到 JS；比如 iOS 官方 App 喜欢底部 tabbar 导航，而安卓官方喜欢左侧的抽屉式导航(个人还是想吐槽一下这个低效的设计)...

## 所改即所见

传统的客户端开发，每次改动都要重新编译和构建。即使是调一个 UI 视图的几个像素，也要等那么久。而不像 web 开发那样，保存一下文件，刷新一下游览器就可以了。

而到了 RN，强大的调试工具使得开发效率大幅提升。还是同样调整几个像素，我也只要保存一下 JS 文件然后 CMD+R 一下模拟器就能马上看到我改动的东西了。no more compile and build again!

chrome 也提供了很方便的 JS 代码 debugger。

## 描述性布局

对于一个动态变化的界面，往往会有 add，remove，move 等对 subview 的操作，这些操作可能散布在某个文件各个片段或者某些文件的各个片段。这些代码如果没有好好管理好，或者约定好，对于开发人员来说，尤其对于接手的新成员来说，并不是那么一目了然，业务复杂点奇葩点，那么维护起来是有些蛋疼的。

而到了 RN，每个组件只有一个方法(`render`)用来描述这个组件长什么样。没有了对视图的 add，remove，move 的操作。每个组件在`render`里根据自身的不同内部状态来决定自己该有什么组件组成。 并且 FB 通过 JSX 语法给开发人员了一种描述性布局体验。

通过'内联 css '来做到对 UI 的啰嗦代码进行分离。布局方案采用 css3 的 flexbox，无论在 web，还是在 iOS 和 android，都是 learn once，write anywhere。

## Re-render everything

所有与 view 绑定的数据变化时，不用再一个个去找这些数据将要影响的那些 view 的布局或者展示，只需要将当前组件重新刷新一下就行了。react 强大的 virtual dom diff 算法会帮我搞定哪些子 view 需要重新刷新，哪些又不需要。

## 快速发布(codePush)

对于发布和托管 app 的 js bundle，我选择了巨硬的[code-push](http://microsoft.github.io/code-push/)。有几个点还不错：

1. 免费的代码存储服务，cdn 下载服务.（至少现在 beta 版是免费的）
2. 配套的统一的 JS 接口，iOS 和 android 各自 native 端下载更新的实现。完善的 CLI 工具。
3. 提供`A/B test`的发布功能。

## 有些不足

1. 版本更新不兼容。
2. ListView 多图长列表内存消耗还是大。

## RN技术栈与工具

- flexbox 布局
- ES6 & ES5, 尤其是 promise
- react 了解

## 总结
快速开发，快速发布，又可以兼容两个平台。native 开发真的要失业了吗？

## 参考资料

- [react-native-bringing-modern-web-techniques-to-mobile](https://code.facebook.com/posts/1014532261909640/react-native-bringing-modern-web-techniques-to-mobile/)--[中文版](http://gold.xitu.io/entry/5693765a60b27e9ba8e055cf)
- [code-push](http://microsoft.github.io/code-push/)
- [ListView performance](https://github.com/facebook/react-native/issues/499)
- [React Architecture](https://code.facebook.com/videos/1507312032855537/oscon-2014-react-s-architecture/)
