---
title: New instaFollow
date: 2013-08-28 17:53:30
---


之前的版本用GCD来些API，dispatch_group, dispatch_anything...

我之前可能提到过，GCD做网络最大的缺点就是没有取消机制。之前的，你一旦开始一个操作，那就是永不停息的野马了。要让野马回头，可以的做法就是给野马发个通知，当然野马要始终观测通知内容值的变化。你可能感觉到，对操作内部有观测值的东西就是NSOperation了。

我好像说过好多次了，NSOperation是一个很牛B的东西。一般情况下，你是不需要继承NSoperation的concurrent模式，那为什么AFNetworking和SDWebImage都这么做了呢? 是不是因为他们都用到了NSURLConnection的delegate呢?

就我现在的菜鸟水平，我只能在operation的main里面做同步的操作。可能是因为他们都需要- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data
监听这个方法，所以才搞成了concurrent的operation。

现在突然干不动活了。今天做了好多事情，鸡血状态已经没有了。

现在我是注释帝，xcode支持这种格式的注释

	/*!
	 @method otherUserDownloadMeidaWithCursor:userId:compeletionHandler:
	 @abstract Download other user media with cursor number to return
	 @discussion Why our callback bring back a cursor, it is for your loadmore fucntion
	 you should set a string property to hold the latest cursor
	 @param cursor the last cursor
	 @param userId other user id
	 @param customHandler to hold the callback
	 @result void
	 */
	- (void)otherUserDownloadMeidaWithCursor:(NSString *)cursor
	                                  userId:(NSString *)userId
	                      compeletionHandler:(NoRecursiveResultBlock)handler;


效果图:

![anyStr](/images/zhushidi.png "像不像真的？")
