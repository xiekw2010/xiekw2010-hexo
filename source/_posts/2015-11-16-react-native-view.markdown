---
layout: post
keywords: blog
description: blog
title: "React Native View布局"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-11-16 17:53:30
---

React-Native 之布局UIKit(iOS)

本文探讨一下React-Native(以下简称RN)iOS端的布局过程。

这个过程包括RN从App(模块)启动到最后JSX的View映射成UIKit的View渲染。这里我把这个过程分为两步，第一步加载rootView，第二步rootView加载JSX的view

先看第一步加载RootView：

![开始](../images/iOS beginRender.png)

第一步RootView通常会在程序里这么初始化的。

```ruby
  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"RN_UI"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];

```

1. 这里用`RootView`做为一个容器，可以理解为`webView`加载网络资源`jsCodeLocation`。
2. 初始化JS与OC的方法表，这方法表可以理解为他们通信的桥梁。具体原理可以看一下[bang的这篇博客](blog.cnbang.net/tech/2698/)。
3. 初始化第二部的桥梁，加载OC的本地模块并告诉JS模块
4. 同时UI上显示一个Loading 远程资源的状态。
5. 等到环境构建完后(远程资源加载到本地内存，JS加载完OC本地的模块)
6. 初始话rootView的contentView，在`RCTUIManager`里设置它作为`rootShadowView`，关于`shadowView`后面会再作介绍，这里暂且当它为一个普通的`UIView`好了。这里面的UI更新频率是根据`CADisplayLink`的频率来对JS做`batchUpdate`的。
7. `RCTRootView`通知JS运行JS程序。
8. 开始渲染`index.ios.js`里的入口程序。
9. `RCTRootView`停止LoadingView状态，后续内容的展现全是JS的程序逻辑了。

第二步，UIView怎么通过JSX来实现布局的。

先来埋个伏笔：React-Native是用css3的Flexbox来实现布局的，UIKit要不就是绝对布局(`setFrame`)要不就是`autoLayout`，最后UIKit是怎么用的JS里的`Flexbox`呢？

来看一个简单的例子:

```ruby
<View style={{flexDirection: 'column'}}>
  {shopItem.isTopFloor && CommonComponents.renderFloorHeader(shopItem.floorName)}
  <View style={styles.floorDoubleItemContainer}>
    <SmallShopComponent shopItem={leftItem} />
    <View style={{width: 5}}></View>
    <SmallShopComponent shopItem={rightItem} />
  </View>
</View>

```
这里的语法和HTML很像，只不过Dom节点变成了react似的虚拟Dom节点样式。

看看JS里的`View`是怎么实现的？

```ruby
var RCTView = createReactNativeComponentClass({
  validAttributes: ReactNativeViewAttributes.RCTView,
  uiViewClassName: 'RCTView',
});
```
这里的`createReactNativeComponentClass`通过调用堆栈最后是调用到`ReactNativeBaseComponent`的`mountComponent`。这里有个很关键的`RCTUIManager.createView`。

`RCTUIManager`是什么？它OC本地的一个的桥接模块，这个模块初始化的时候会把所有`RCTViewManager`的子类都收集起来放到一个`_componentDataByName`里

JS里面每个特定的基础`View`模块(比如`MapView`, `WebView`)一般都是由OC里的某个特定的`ViewManager`桥接过来的，如果不是，那也是这些桥接的`View`的组合(比如`ListView`)。

重点来了，当JS调用OC的`createView:viewName:rootTag:props:`后，OC本地会创建一个`RCTShadowView`。

`RCTShadowView`看它的基类，它并非一个UIView，而是一个结合`layout.c`的，这个`layout.c`文件是`Flexbox`布局算法的c语言实现。

最后从js传来的Flexbox属性会交给这个`shadowView`的`css_node`处理。在前面第一步`RootView`初始话的第六步里，`batchUpdate`里，在主线程把所有的`shadowView`flexbox模型的计算结果布局到真正的`View`上面。

大致流程是这样的。

![](../images/jsafterRender.png)
