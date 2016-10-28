---
title: A image downloader
date: 2013-09-03 17:53:30
---

应用场景，屏幕上只有一个Imageview，你只有一堆图片的url地址。需求：有个按钮点了以后，要快速切换到下一张图片。

这时候用SDWebImage的"UImageView+WebCache"是不是感觉有点捉急了，他TM还要一张张的加载。这里我们要的效果是，我在看某张图片的时候，后面的几张也给我TM加载好，就算没有加载好，我切过去的时候也给我显示一个进度条啊。

我的ImageDownloader的思路是：

1、既然有一组url地址，那就先给我吧，我直接去开始加载了。针对每个operation，加载完后就把value:图片 key:url地址 存到我这个singleton的NSCache里

	- (void)startDownloadImagesWithURLArray:(NSArray *)urlstrArray
	{
	    for (NSString *str in urlstrArray) {
	        if (str.length > 0) {
	            [self enqueueOperationWithStr:str];
	        }
	    }
	}

	- (void)enqueueOperationWithStr:(NSString *)str
	{
	    NSMutableURLRequest *req = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:str] cachePolicy:NSURLCacheStorageAllowed timeoutInterval:8.0];
	    ILSMLWebImageDownloaderOperation *op = [[ILSMLWebImageDownloaderOperation alloc] initWithRequest:req queue:self.workingQueue];
	    [self.downloadQueue addOperation:op];

	    op.completedBlock = ^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
	        if (!error) {
	            MYLog(@"ILSMLImageDownloader is cache image for url %@", str);
	            [self.dlCache setObject:image forKey:str];
	        }else {
	            MYLog(@"ILSMLWebImageDownloaderOperation download image error occured %@", error);
	        }
	    };

	    if (!self.blocksDic[str]) {
	        self.blocksDic[str] = op;
	    }
	}

2、外部接口就提供，你直接根据你的url地址来获取我这个singleton里cache的图片吧，如果我还没cache吧，那么我就告诉你，我现在的进度progressBlock，另外，一但我加载完，我也会告诉你的哦，compeletionBlock

	- (UIImage *)imageForUrl:(NSString *)imageURLStr progressBlock:(SDWebImageDownloaderProgressBlock)progressBlock compeletionBlock:(SDWebImageDownloaderCompletedBlock)compeletionBlock
	{
	    UIImage *cacheImage = [self.dlCache objectForKey:imageURLStr];
	    if (!cacheImage) {
	        ILSMLWebImageDownloaderOperation *op = self.blocksDic[imageURLStr];
	        if (op) {
	            if (progressBlock) {
	                op.progressBlock = ^(NSUInteger receivedSize, long long expectedSize) {
	                    progressBlock(receivedSize, expectedSize);
	                };
	            }

	            if (compeletionBlock) {
	                op.completedBlock = ^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
	                    if (!error) {
	                        [self.dlCache setObject:image forKey:imageURLStr];
	                    }
	                compeletionBlock(image, data, error, finished);
	                };
	            }
	        }else {
	            MYLog(@"ILSMLImageDownloader operation block does not exist");
	        }
	    }
	    return cacheImage;
	}

3、做男人当然要做一个有责任心的男人，既然你和它没关系了，那let it free吧。所以第三个接口就是清理掉不用的image

	- (void)removeImageWithURLStr:(NSString *)imageURLStr
	{
	    ILSMLWebImageDownloaderOperation *op = self.blocksDic[imageURLStr];
	    [op cancel];
	    [self.blocksDic removeObjectForKey:imageURLStr];
	    [self.dlCache removeObjectForKey:imageURLStr];
	}


具体的使用例子：

	- (void)loadImageData
	{
	    if (self.mediaDicArray.count == 0) {
	        [[ILSINSNewInsatgramApi sharedInstance] otherUserDownloadMeidaWithCursor:self.currentCusor
	                                                                          userId:@"203560" compeletionHandler:^(NSArray *mediaDicArray, NSString *nextCusor, NSError *error)
	         {

	             if (!error) {
	                 [self.mediaDicArray addObjectsFromArray:mediaDicArray];

	                 NSMutableArray *mArray = [NSMutableArray arrayWithCapacity:mediaDicArray.count];
	                 for (id obj in mediaDicArray) {
	                     [mArray addObject:[obj valueForKeyPath:@"images.low_resolution.url"]];
	                 }
	                 [[ILSMLImageDownloader sharedDownloader] startDownloadImagesWithURLArray:mArray];

	                 self.currentCusor = nextCusor;
	                 [self customImage];
	             }
	         }];
	    }else {
	        [self customImage];
	    }
	}

	- (void)customImage
	{
	    NSString *path = [self.mediaDicArray[0] valueForKeyPath:@"images.low_resolution.url"];

	    UIImage *result = [[ILSMLImageDownloader sharedDownloader] imageForUrl:path progressBlock:^(NSUInteger receivedSize, long long expectedSize) {
	        MYLog(@"-----ILSMLEarnCoinsViewController is loading is %d  total is %llu", receivedSize, expectedSize);
	    } compeletionBlock:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
	        MYLog(@"-----ILSMLEarnCoinsViewController is compeletion");
	        self.currentImageView.image = image;
	    }];

	    self.currentImageView.image = result;
	}

	- (void)skip:(id)sender
	{
	    if (self.mediaDicArray.count > 0) {
	        NSString *path = [self.mediaDicArray[0] valueForKeyPath:@"images.low_resolution.url"];
	        [[ILSMLImageDownloader sharedDownloader] removeImageWithURLStr:path];
	        [self.mediaDicArray removeObjectAtIndex:0];
	    }
	    [self loadImageData];
	}


记录一下最近吧，昨天早上被指派了一个新项目，自己iOS端逻辑今天全弄好了（过段时间整理下Instagram objc的API，open source出来），下午贴图。再过几天，等服务器端同事接口写好，我再花个两天，测试再花个几天，下周估计就能上线了。这个鸡巴项目叫morelikers。我们app扮演了一个”拉皮条“的角色。

下下周，中秋， lonely alone 去北京感受一下。
