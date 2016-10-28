---
layout: post
keywords: blog
description: blog
title: "JS to OC(1), H5 bridge"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-10-17 17:53:30
---

###Javascript 与 Objective-C 互相通信（1）

通过Javascript(以下简称JS)与Objective-C(以下简称OC)的通信，再加上JS脚本的动态一些下发，我们的app就能变得很灵活。加一个新功能，修复一个bug，开发一个新模块页面，不用再等app store的审核了。

这个系列的文章将介绍目前比较通用的几种方式，h5的[JSBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)，[Apple 原生的JavascriptCore](https://developer.apple.com/library/mac/documentation/Carbon/Reference/WebKit_JavaScriptCore_Ref/)，[React-Native](http://facebook.github.io/react-native/)，[JSPatch](https://github.com/bang590/JSPatch)。

#####h5 JSBridge

在webview环境下，我们可以使用OC方法`stringByEvaluatingJavaScriptFromString:`，来调用在当前webview的JS上下文中的JS脚本的运行结果。这就完成了OC调用JS的过程。

反之，JS如何调用OC方法呢？目前的大致流程:

![image](../images/js2oc/1.png)

在webview如何做到这个需求呢？webView有个拦截页面刷新的回调接口`webView:shouldStartLoadWithRequest:navigationType:`。

在web里，我们可以调用JS来通知页面发一个傀儡请求，比如说这个请求是这样的`fakeScheme://fakeHost`，然后在这个拦截回调里拦截到这个请求，根据这个请求的`scheme`和`host`规则或者`path`,`query`这些字符串来映射反射调用native的方法，最后native方法在通过`stringByEvaluatingJavaScriptFromString:`来回调给js。

大致原理就这样，因为OC与JS的callback不能互传，所以这里会牵涉到一些callback互传的trick。下面就来封装一个JS<=>OC的接口友好的Bridge.
	// OC端接口

	@interface OCJSBridge: NSObject<UIWebViewDelegate>

	typedef void (^JSCallback)(id responseData);
	typedef void (^OCToJSCallback)(id data, JSCallback responseCallback);

	// OC发消息给JS, 并接受JS处理消息后的回调。
	- (void)sendToJSMessage:(id)message responseJSCallback:(JSCallBack)callback;

	// OC注册JS的调用方法和数据参数，OC处理数据后，在回调传给JS
	- (void)registerJSModuleName:(NSString *)handleName responseOCCallback:(OCToJSCallback)callback


	// JS端接口
	// JS发送消息给OC, 并接受OC处理消息后的回调。
	function send(moduleName, data, responseCallback) {
	}

	// JS注册OC的调用方法
	function registerHandler(handlerName, handler) {
	}

	Window.JSBridge = {
		send: send,
		registerHandler: registerHandler,
	}

以数据库查询为例子，JS需要调用OC的SQL模块，OC注册这个JS调用方法，并根据JS提供的参数做一些本地操作然后把最后数据回传给JS。

JS => OC，JS调用`JSBridge.send('SQL', {table: 'JHS'}, responseCallback)`方法, OC本地做

	[_ocJSBridge registerJSModuleName:@"SQL" responseOCCallback:^(id data, JSCallback responseCallback) {
        NSLog(@"testObjcCallback called: %@", data);
        id ocHandledData = [SQLManager syncQueryWith:[NSString stringWithFormat:@"SELECT * from %@", data.table]];
        responseCallback(ocHandledData);
    }];

调用过程如图所示:

![image](../images/js2oc/2.png)

#####总结

1. OC调用JS，就用`stringByEvaluatingJavaScriptFromString:`
2. JS调用OC，OC里的实现原理就是拦截请求的URLRequest，根据与html约定的host与scheme来触发bridge注册JS模块的block执行。
3. 执行完block后，再用`stringByEvaluatingJavaScriptFromString:`把JS的callbackId回传给JS，然后JS执行JS的回调。
