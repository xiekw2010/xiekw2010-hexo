---
title: Great NSOperation
date: 2013-07-01 17:53:30
tags:
---

As we all know, the great AFNetworking and the SDWebImage is written based on the Concurrent NSOperation.

While, to subclass the NSOperation is a little more tricky(It has two style, noconcurrent and concurrent). So, in most concurrent environment, we will use the GCD style instead of it.(dispatch async some queue).

But as we see, the most weakness of GCD is that it's hard to support cancelling the operation which will leads many waste.

So when it comes to a little more tricky operation, the nsoperation will save you. And its greatness is the simple of API. Just:

	NSOperationQueue *queue = [[NSOperationQueue alloc] init];
	[queue addOperationWithBlock:^{
	    NSArray *someDataArray = [SomeNetWorkAPI fetchSomeDataArray];
	    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
	        self.tableViewDataArray = someDataArray;
	        [self.tableView reloadData];
	    }];
	}];

You see, it is like dipatchAsync() playmode. And the goodness is when you dont need those netNetWorkEvent, you could just [_instanceOperationQueue cancelAllOperetions]; 这个方法会取消queue里还没被执行到的operation，但是如果有个operation正在执行，那它就默认不会被取消。除非....
你这个operation里面有自我认识到自己被取消后的动作。比如说，

    NSBlockOperation *bop = [[NSBlockOperation alloc] init];
    [bop addExecutionBlock:^{
        for (int i = 0; i < 100000; i ++) {
            [self processData:i];
        }
    }];
    [self.opQueue addOperation:bop];

这里有个大循环，是不是最好当得知这个operation 被cancel的时候，跳出循环处理呢？那么，一个operation如何得知自己被取消了呢？ [bop isCancelled];

所以这里最好,

    NSBlockOperation *bop = [[NSBlockOperation alloc] init];
    [bop addExecutionBlock:^{
        for (int i = 0; i < 100000; i ++) {
        	if ([bop isCancelled]) return;
            [self processData:i];
        }
    }];
    [self.opQueue addOperation:bop];

    // 注意在ARC下，由于block的引用循环，所以这里要这样写

    NSBlockOperation *bop = [[NSBlockOperation alloc] init];
    __weak NSBlockOperation *wbop = bop;
    __weak SomeVC *wself = self;
    [bop addExecutionBlock:^{
        for (int i = 0; i < 100000; i ++) {
        	if ([wbop isCancelled]) return;
            [wself processData:i];
        }
    }];
    [self.opQueue addOperation:bop];

当然具体到自己写的Operation，那就要在subclass内部辨认了，这里只是简单的小玩意。

不知道大家有没有碰到这么种情况，在tableview滚动的时候，setCellImageView如果用异步的方法，那么滚动的时候，那么imageview会乱跳，前面的图片被用到后面去了。这是tableview cell的reuse机制外加你是异步获取图片而导致的。（前面获取的图片获取到的时候正好滚动到后面去了，后面的就用前面的了）
这里可以尝试用[tableView cellForRowAtIndexPath:indexPath].imageView.image = callBackImage;来定位当前的Cell。

To be continued...
