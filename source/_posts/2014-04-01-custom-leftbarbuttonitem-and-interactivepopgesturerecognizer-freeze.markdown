---
layout: post
keywords: blog
description: blog
title: "Custom leftBarButtonItem and interactivePopGestureRecognizer Freeze"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2014-04-01 17:53:30
---

iOS7有个系统app都有个新功能，支持手势返回。也就是navigationViewController.interactivePopGestureRecognizer

如果你乖乖的用系统自带的UIBarButtonItem来定义navigationViewController的leftBarButtonItem的话，那么一切都ok。但是如果你是通过initWithCustomView:来定义leftBarButtonItemd的话，你需要在你父ViewController的ViewDidAppear方法里手动开启它, 当然顺便请设置好Gesture的delegate.

	- (void)viewDidAppear:(BOOL)animated
	{
	    [super viewDidAppear:animated];
	    if ([self.navigationController respondsToSelector:@selector(interactivePopGestureRecognizer)]) {
	        self.navigationController.interactivePopGestureRecognizer.delegate = self;
	    }
	}

可是，可是，这样做还不够。你可以快速pop一个viewcontroller并且快速手势返回，这时候，你的整个屏幕就freeze了。

怎么办呢？

看了网上的整体思路就是，你当pop的时候，把navigationController.interactivePopGestureRecognizer.enabled设为NO，当你navigationController:didShowViewController:animated:（nav的delegate）时候再打开。

具体到代码就是：

1. 在navigationViewController里

	- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated
	{
	    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
	        self.interactivePopGestureRecognizer.enabled = NO;    

	    [super pushViewController:viewController animated:animated];
	}

2. 在你ViewController里

	- (void)navigationController:(UINavigationController *)navigationController
	       didShowViewController:(UIViewController *)viewController
	                    animated:(BOOL)animate
	{	    
	    if ([self.navigationController respondsToSelector:@selector(interactivePopGestureRecognizer)])
	        self.navigationController.interactivePopGestureRecognizer.enabled = YES;
	}


问题来了，如何令所有的navigationViewController都能有我们经过处理后的pushViewController:animated:方法。可以看到，这里我是subclass了。

这里我主要想介绍另外一种objc runtime的方法， method_swizzle。

	#import <objc/runtime.h>

	@implementation UINavigationController (PopGesture)

	+ (void)load {
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        Class class = [self class];

	        // When swizzling a class method, use the following:
	        // Class class = object_getClass((id)self);

	        SEL originalSelector = @selector(pushViewController:animated:);
	        SEL swizzledSelector = @selector(xxx_pushViewController:animated:);

	        Method originalMethod = class_getInstanceMethod(class, originalSelector);
	        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

	        BOOL didAddMethod =
	        class_addMethod(class,
	                        originalSelector,
	                        method_getImplementation(swizzledMethod),
	                        method_getTypeEncoding(swizzledMethod));

	        if (didAddMethod) {
	            class_replaceMethod(class,
	                                swizzledSelector,
	                                method_getImplementation(originalMethod),
	                                method_getTypeEncoding(originalMethod));
	        } else {
	            method_exchangeImplementations(originalMethod, swizzledMethod);
	        }
	    });
	}

	#pragma mark - Method Swizzling

	- (void)xxx_pushViewController:(UIViewController *)viewController animated:(BOOL)animated
	{
	    NSLog(@"xxx_pushViewController: %@", self);
	    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
	        self.interactivePopGestureRecognizer.enabled = NO;

	    [self xxx_pushViewController:viewController animated:animated];
	}

	@end

这样就不要subclass了。
