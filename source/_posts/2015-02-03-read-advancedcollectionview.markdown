---
layout: post
keywords: blog
description: blog
title: "Read AdvancedCollectionView"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-02-03 17:53:30
---

WWDC2014年AdvancedCollectionView session的源代码是目前为止我见过最难懂的代码(难懂是因为我修为还太少)，据说这是Apple 官方App iTunes connnect的源代码的一部分。

这里说说里面用到的OSAtomicCompareAndSwap32方法吧。个人的理解，这是一个线程安全的布尔值开关。具体的使用例子:

	//_cancellationPredicate是一个int32_t的instance variable

	if (OSAtomicCompareAndSwap32(0, 1, &_cancellationPredicate)) {

        // Make sure we don't remove ourself before addObserver: completes
        if (_options & NSKeyValueObservingOptionInitial) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [_observee removeObserver:self forKeyPath:_keyPath];
                _observee = nil;
            });
        }
        else {
            [_observee removeObserver:self forKeyPath:_keyPath];
            _observee = nil;
        }
    }

这段代码确保了If里面的执行线程安全的只执行了一次，这里的意思也就是这个类实例只removeObserver:self 一次。

那么什么是CompareAndSwap呢？详细可以参考Mike Ash的[介绍](https://www.mikeash.com/pyblog/friday-qa-2011-03-04-a-tour-of-osatomic.html)

大概的方法实现是这样的:

	 bool CompareAndSwap(int old, int new, int *value) {
	        if(*value == old)
	        {
	            *value = new;
	            return true;
	        }
	        else
	            return false;
	    }

意思是比较value指针指的对象的值与old是否相同，如果相同返回true并且更新*value到new，反之则返回false。对照一下上面的实现，我们不难看出这不就是个不二开关吧。如果改了，那就滚，如果没改，那就改一下，并且把标志设为改了。

一般人的思路(我)觉得布尔值的开关控制就够了，不用管可能复杂的线程安全了。这里面的OSAtomic操作顾及到了这一点。如果以后涉及到线程安全的布尔开关操作，我们可以用一下这个方法。

更多应用例子:

1. 确保block只执行一次。

		__block int32_t complete = 0;

		[self aapl_addObserverForKeyPath:@"loadingComplete" 					options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew 					withBlock:^(id obj, NSDictionary *change, id observer) {

	        BOOL loadingComplete = [change[NSKeyValueChangeNewKey] boolValue];
	        if (!loadingComplete)
	            return;

	        [self aapl_removeObserver:observer];

	        // Already called the completion handler
	        if (!OSAtomicCompareAndSwap32(0, 1, &complete))
	            return;

        	block();
        }];

2. 作为一个开关来使用, 确保了当前情况下只有一个block在执行并且等到这个block()结束后才接受后面的block()。

		void aapl_debounce(int32_t *predicate, dispatch_block_t block)
		{
		    if (OSAtomicCompareAndSwap32(0, 1, predicate)) {
		        block();
		        OSAtomicDecrement32(predicate);
		    }
		}


这份源代码还有很多值得研究的地方。里面的设计模式(CollectionView的datasource, stateMachine, 高级使用CollectionViewLayout等)。
