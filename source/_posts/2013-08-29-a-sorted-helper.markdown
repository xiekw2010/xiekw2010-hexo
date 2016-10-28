---
title: A sorted helper
date: 2013-08-29 17:53:30
---

假设有个这样的Dictionary,叫numDic:

	@{@"a" : @(1), @"b" : @(2), @"dd" : @(4), @"pp" : @(0)}

现在要对其每个key对应的value排序，也就是对1，2，3排序(假设升序), 结果应该是:

	@[@"pp", @"a", @"b", @"dd"]

现在你需要的只是一个对象假设叫SortedHelper

	- (id)initWithKey:(NSString *)skey withCount:(NSInteger)scount
	{
	    if (self = [super init]) {
	        self.key = skey;
	        self.count = scount;
	    }
	    return self;
	}

	- (NSComparisonResult)myCompare:(SortDictHelper *)helper
	{
	    return self.count >= helper.count ? NSOrderedAscending : NSOrderedDescending;
	}

	- (NSComparisonResult)yourCompare:(SortDictHelper *)helper;
	{
	    return self.count >= helper.count ? NSOrderedDescending : NSOrderedAscending;
	}

	+ (NSArray *)sortedDecending:(BOOL)decending mapDic:(NSDictionary *)dic
	{
	    NSMutableArray *mArray = [NSMutableArray arrayWithCapacity:dic.count];
	    [dic enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
	        if ([obj isKindOfClass:[NSArray class]]) {
	            NSArray *array = obj;
	            SortDictHelper *helper = [[SortDictHelper alloc] initWithKey:key withCount:array.count];
	            [mArray addObject:helper];
	        }else if ([obj isKindOfClass:[NSNumber class]]){
	            NSNumber *number = obj;
	            SortDictHelper *helper = [[SortDictHelper alloc] initWithKey:key withCount:[number integerValue]];
	            [mArray addObject:helper];
	        }
	    }];

	    if(decending)
	    	[mArray sortUsingSelector:@selector(myCompare:)];
	    else
	    	[mArray sortUsingSelector:@selector(yourCompare:)]

	    NSMutableArray *keyArray = [NSMutableArray arrayWithCapacity:dic.count];
	    [mArray enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
	        SortDictHelper *helper = obj;
	        [keyArray addObject:helper.key];
	    }];
	    return keyArray;
	}

使用就是:

	NSArray *result = [SortedHelper sortedDecending:NO mapDic:numDic];

当然这里也对:

	@{@"a" : @[@"cc", @"dd", @"ee"], @"b" : @[@"00", @"33"], @"pp" : @[@"1"]}

这种数据结构做了扩展。
