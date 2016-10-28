---
layout: post
keywords: blog
description: blog
title: "React Native通信过程"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-10-31 17:53:30
---

####Javascript 与 Objective-C 通信(2)
---

上期讲了H5的JSBridge通信机制，这期讲讲React-Native的通信机制。

目前React-Native做到的效果是这样的。

![image](../images/js2oc/1.png)

最终可以实现到如下两点:

1. OC有的类, JS也有。
2. 调用JS就像调用OC效果一样。

它是如何做到的呢？我们先从程序入口开始看。

#####程序入口
---

```ruby
// step1
  jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/index.ios.bundle"];
// step2
  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"RN_CNNode"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];

  UIViewController *rootViewController = [[UIViewController alloc] init];
// step3
  rootViewController.view = rootView;
```

这段代码有什么效果呢？

1. `rootViewController`的内容显示将由`RCTRootView`来负责。
2. `RCTRootView`内容将由`jsCodeLocation`和`RN_CNNode`这个模块名来决定。
3. `jsCodeLocation`可以是一个网络资源。

打开`jsCodeLocation`的文件，这是一个js执行文件。也就是我们写的所有js文件的打包结果。这里有个[例子](http://wapp.waptest.taobao.com/rct/jutry/mainjsbundle.js)。

#####模块配置表
---

如果js要调用oc的方法，那么js首先要知道oc有什么方法。所以在上面`RCTRootView`初始化的时候，oc要生成一个模块配置表传给js。那么这里oc是如何生成模块配置表的呢？

在oc的`RCTBatchBridge`里做了如下事情:

```ruby
// 加载远程js执行文件到本地内存中。
[self loadSource:^(__unused NSError *error, NSString *source) {
    sourceCode = source;
}];

// 初始化本地模块列表
[self initModules];

// 设置执行js的虚拟机
[self setupExecutor];

// 打包本地的模块列表
config = [weakSelf moduleConfig];

// 用js虚拟机把本地模块列表传给js上下文。
// 把这个config保存在js的一个`__fbBatchedBridgeConfig`全局变量里
[weakSelf injectJSONConfiguration:config onComplete:^(__unused NSError *error) {}];

// 把js执行线程加到runloop中去。      
[weakSelf executeSourceCode:sourceCode];
```

在js的`MessageQueue`负责接受上面的全局变量，并在js端生成一个模块配置表。打印出来如下：

	"RCTImageEditingManager": {
		"methods": {
		  "cropImage": {
		    "type": "remote",
		    "methodID": 0
		  }
		},
		"moduleID": 2
	},

最后，js端和oc都保留了这份配置表，js在调用oc方式时，通过bridge里的配置表把模块方法转为模块ID和方法ID传给oc，oc通过bridge的模块配置表找到对应的方法执行之，以上述代码为例，流程大概是这样（先不考虑callback）：

![2](../images/js2oc/2.jpeg)

#####自动生成模块配置表
---
首先oc每个暴露给js的类要满足`@protocol(RCTBridgeModule)`协议。在implementation(.m文件里)需要写入这个宏`RCT_EXPORT_MODULE()`，这个宏做如下事情:

```ruby
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
```

结合前面`RCTBridge`里的`initialize`方法来看：

```ruby
static unsigned int classCount;
Class *classes = objc_copyClassList(&classCount);

for (unsigned int i = 0; i < classCount; i++)
{
  Class cls = classes[i];
  Class superclass = cls;
  while (superclass)
  {
    if (class_conformsToProtocol(superclass, @protocol(RCTBridgeModule)))
    {
      if (![RCTModuleClasses containsObject:cls]) {
        RCTLogWarn(@"Class %@ was not exported. Did you forget to use "
                   "RCT_EXPORT_MODULE()?", cls);

        RCTRegisterModule(cls);
        objc_setAssociatedObject(cls, &RCTBridgeModuleClassIsRegistered,
                                 @NO, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
      }
      break;
    }
    superclass = class_getSuperclass(superclass);
  }
}
free(classes);
```
从这里看出，oc代码在加载时就已经把所有满足`RCTBridgeModule`协议的类都提取出来了，并把这些类在程序初始化js上下文后(参见`RCTBatchBridge`)再解析其中暴露给JS的方法后做成配置表再传给js上下文。具体保存在`NativeModules`这个模块里。有兴趣的可以看一下RN源码的`AlertIOS.js`和`RCTAlertManager.m`。

#####调用流程
---
接下来看看JS调用OC模块方法的详细流程，包括callback回调。这时需要细化一下上述的调用流程图：
![](../images/js2oc/ReactNative2.png)

看起来有点复杂，不过一步步说明，应该很容易弄清楚整个流程，图中每个流程都标了序号，从发起调用到执行回调总共有11个步骤，详细说明下这些步骤：

1. JS端调用某个OC模块暴露出来的方法。
2. 把上一步的调用分解为ModuleName,MethodName,arguments，再扔给MessageQueue处理。
在初始化时模块配置表上的每一个模块都生成了对应的remoteModule对象，对象里也生成了跟模块配置表里一一对应的方法，这些方法里可以拿到自身的模块名，方法名，并对callback进行一些处理，再移交给MessageQueue。具体实现在MessageQueue.js的`_genModules`里。

3. 在这一步把JS的callback函数缓存在MessageQueue的一个成员变量里，用CallbackID代表callback。在通过保存在MessageQueue的模块配置表把上一步传进来的ModuleName和MethodName转为ModuleID和MethodID。

4. 把上述步骤得到的ModuleID,MethodId,CallbackID和其他参数argus传给OC。至于具体是怎么传的，后面再说。

5. OC接收到消息，通过模块配置表拿到对应的模块和方法。
实际上模块配置表已经经过处理了，跟JS一样，在初始化时OC也对模块配置表上的每一个模块生成了对应的实例并缓存起来，模块上的每一个方法也都生成了对应的RCTModuleMethod对象，这里通过ModuleID和MethodID取到对应的Module实例和RCTModuleMethod实例进行调用。具体实现在_handleRequestNumber:moduleID:methodID:params:。

6. RCTModuleMethod对JS传过来的每一个参数进行处理。
RCTModuleMethod可以拿到OC要调用的目标方法的每个参数类型，处理JS类型到目标类型的转换，所有JS传过来的数字都是NSNumber，这里会转成对应的int/long/double等类型，更重要的是会为block类型参数的生成一个block。
例如-(void)select:(int)index response:(RCTResponseSenderBlock)callback 这个方法，拿到两个参数的类型为int,block，JS传过来的两个参数类型是NSNumber,NSString(CallbackID)，这时会把NSNumber转为int，NSString(CallbackID)转为一个block，block的内容是把回调的值和CallbackID传回给JS。
这些参数组装完毕后，通过NSInvocation动态调用相应的OC模块方法。
7. OC模块方法调用完，执行block回调。
8. 调用到第6步说明的RCTModuleMethod生成的block。
9. block里带着CallbackID和block传过来的参数去调JS里MessageQueue的方法invokeCallbackAndReturnFlushedQueue。
10. MessageQueue通过CallbackID找到相应的JS callback方法。
11. 调用callback方法，并把OC带过来的参数一起传过去，完成回调。
整个流程就是这样，简单概括下，差不多就是：JS函数调用转ModuleID/MethodID -> callback转CallbackID -> OC根据ID拿到方法 -> 处理参数 -> 调用OC方法 -> 回调CallbackID -> JS通过CallbackID拿到callback执行。

#####事件响应
---

上述第4步留下一个问题，JS是怎样把数据传给OC，让OC去调相应方法的？

答案是通过返回值。JS不会主动传递数据给OC，在调OC方法时，会在上述第4步把ModuleID,MethodID等数据加到一个队列里，等OC过来调JS的任意方法时，再把这个队列返回给OC，此时OC再执行这个队列里要调用的方法。

一开始不明白，设计成JS无法直接调用OC，需要在OC去调JS时才通过返回值触发调用，整个程序还能跑得通吗。后来想想纯native开发里的事件响应机制，就有点理解了。native开发里，什么时候会执行代码？只在有事件触发的时候，这个事件可以是启动事件，触摸事件，timer事件，系统事件，回调事件。而在React Native里，这些事件发生时OC都会调用JS相应的模块方法去处理，处理完这些事件后再执行JS想让OC执行的方法，而没有事件发生的时候，是不会执行任何代码的，这跟native开发里事件响应机制是一致的。

说到OC调用JS，再补充一下，实际上模块配置表除了有上述OC的模块remoteModules外，还保存了JS模块localModules，OC调JS某些模块的方法时，也是通过传递ModuleID和MethodID去调用的，都会走到-enqueueJSCall:args:方法把两个ID和参数传给JS的BatchedBridge.callFunctionReturnFlushedQueue，跟JS调OC原理差不多，就不再赘述了。

#####总结
---
安利一下React-Native的好处吧。

1. 很方便的桥接Native模块类(只要写在它提供的宏里就可以了)，so，JS可以很方便的调用native模块类。
2. 开发过程无需考虑OC的感受，遵从React框架的思想进行纯JS开发就行，剩下的事情React Native帮你处理好了。
3. JS/OC不会频繁通信，只会在事件触发时批量传递，提高效率。


####声明
---
本文中的__调用流程__与__事件响应__摘自@bang的博客[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)。由于这里我讲的是`RN 0.11.4`版本，__调用流程__的第二步原文提到的`BatchedBridgeFactory`已经在我这个版本废弃了，新版本统一使用`MessageQueue.js`。
