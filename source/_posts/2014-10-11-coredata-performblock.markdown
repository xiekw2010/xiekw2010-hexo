---
layout: post
keywords: blog
description: blog
title: "CoreData performBlock"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2014-10-11 17:53:30
---


这一年来都是在用[MagicalRecord](https://github.com/magicalpanda/MagicalRecord)来写iOS客户端的数据库。期间我也写过sqlite的项目，两种数据库框架用下来，就使用感受来说，MagicalRecord写起来更舒服一点, 更适合快速开发移动项目。

MagicalRecord，用的最多就属`saveWithBlock:`或者`saveWithBlockAndWait:`这两个方法了。在这个参数block里面把需要增删改的ManagedObject的操作写在里面就行了。那么它里面是如何做到数据库多线程的呢？

我们知道Context是基于PersistStore来创建的，coredata存储的最后一步，我们是把Context的东西通过coordinator存到Store里去。iOS5.0后，coreData支持创建child context了，所以现在流行的做法是先把child context存到parent里去，再把parent按原来的方式存到store里去。当然把child和parent可以有好几代。好了，问题来了，这个儿子context初始化的时候是不是要指定一种类型呢？

是的，它有三种`NSConfinementConcurrencyType`,`NSPrivateQueueConcurrencyType`,`NSMainQueueConcurrencyType`。因为是多线程操作，第一种从名字上来理解它就是拒绝多线程的，所以我们如果要用多线程，一般都用后面两种。如果没用过，可以看看MagicalRecord是怎么用的。

它有两个重要的static context，一个是`rootContext`(与Store交流的，它是NSPrivateQueueConcurrencyType类型，也就说我们magicalRecord的所有last save操作都是在一个背景线程中做的)， 另外一个是`defaultContext`(它是NSMainQueueConcurrencyType，magicalRecord里的所有child context都是基于它创建的NSPrivateQueueConcurrencyType类型）。那么问题来了，NSMainQueueConcurrencyType 与 NSPrivateQueueConcurrencyType有啥差别？官方文档给出的解释是，他们差不多，当设计UI交互的时候用NSMainQueueConcurrencyType。

我们再来看看`saveWithBlock:`和`saveWithBlockAndWait:`具体做了什么？

	+ (void) saveWithBlock:(void(^)(NSManagedObjectContext *localContext))block completion:(MRSaveCompletionHandler)completion;
	{
	    NSManagedObjectContext *mainContext  = [NSManagedObjectContext MR_defaultContext];
	    NSManagedObjectContext *localContext = [NSManagedObjectContext MR_contextWithParent:mainContext];

	    NSLog(@"saveWithBlock mainContext is %@ and localContext is %@", mainContext, localContext);

	    [localContext performBlock:^{
	        if (block) {
	            block(localContext);
	        }

	        [localContext MR_saveWithOptions:MRSaveParentContexts|MRSaveSynchronously completion:completion];
	    }];
	}

	+ (void) saveWithBlockAndWait:(void(^)(NSManagedObjectContext *localContext))block;
	{
	    NSManagedObjectContext *mainContext  = [NSManagedObjectContext MR_defaultContext];
	    NSManagedObjectContext *localContext = [NSManagedObjectContext MR_contextWithParent:mainContext];

	    NSLog(@"saveWithBlockAndWait thread is main thread %d and thread is %@", [NSThread isMainThread], [NSThread currentThread]);

	    [localContext performBlockAndWait:^{
	        if (block) {
	            block(localContext);
	        }

	        [localContext MR_saveWithOptions:MRSaveParentContexts|MRSaveSynchronously completion:nil];
	    }];
	}

明眼人一看就明白，差别就在于`performBlock`和`performBlockAndWait`。来看官方文档怎么说的。

`performBlock`: Asynchronously performs a given block on the receiver’s queue.

`performBlockAndWait`: Synchronously performs a given block on the receiver’s queue.

做一下解释，我想难免会有人跟我一样对此误解。重点在于如何理解*receiver’s queue*？

可能的理解是，我的context是一个NSPrivateQueueConcurrencyType，所以这是一个背景线程，那么我的`performBlock`是在这个context所在的线程下Asynchronously performs a given block。

按照这种理解，我们试验一下:

		//code a
        [MagicalRecord saveWithBlockAndWait:^(NSManagedObjectContext *localContext) {

            NSLog(@"current thread is main thread %d and thread is %@", [NSThread isMainThread], [NSThread currentThread]);

            //some ManagedObject code
            ...
        }];

        //code b
		[MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {

			NSLog(@"current thread is main thread %d and thread is %@", [NSThread isMainThread], [NSThread currentThread]);

			//some ManagedObject code
			...
		}];

这两个localContext都是NSPrivateQueueConcurrencyType类型，但是结果确实code a是主线程，code b是背景线程。

所以这种理解肯定是有误的。对此，得搞清楚一个概念，context不是一个NSThread或者GCD queue对象，它也并不retain线程的东西。所以这里的*receiver’s queue*不是指context，那它指什么呢？那就是指调用这个方法时，系统当前的queue或者thread。所以`performBlock`是在当前线程下在自动dispatch_async了一个线程，立刻返回；相比`performBlockAndWait`的操作就会阻塞当前线程。所以当我们用`saveWithBlockAndWait`的时候，如果block里面内容复杂，那就会阻塞主线程了，造成页面卡住了。
