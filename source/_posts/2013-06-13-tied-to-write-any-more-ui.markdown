---
title: Tied to write any more UI
date: 2013-06-13 17:53:30
tags:
---

今天早上到现在一直都在写UI。

现在一写到CGRectMake，CGrectGet...就有种想吐的感觉。一到不得不用layoutsubviews的时候就真忧伤啊。

我数数从早上到现在做了几个了，才17个。

晚上2个半小时，我写了5个。

上班的时候，记得有一个卡了好久。

截图下来，我解释下蛋疼在哪里

![anyStr](/images/sb1.png "sb1")

![anyStr](/images/sb2.png "sb2")

![anyStr](/images/sb3.png "sb3")

![anyStr](/images/sb4.png "sb4")

第一眼望去，这么简单的界面。

细看，数字和单位是两种不同字体，也就是说不能用一个button或者label来搞定了，再想一想，这个时间不给它娘一个限制，它是要无法无天的啊（那你这个限制几位数的时候再给它加上呢）。它无法无天了，旁边的小蚯蚓不是要越轨了。另外时间的单位，当单数的时候你不能加S吧。当然下面这些东西要随着单位和时间length来能屈能伸啊。

	- (void)layoutSubviews
	{
	    [super layoutSubviews];

	    CGSize moodSize = [[self.moodButton titleForState:UIControlStateNormal] sizeWithFont:self.moodButton.titleLabel.font];

	    CGFloat moodWidth = 0;
	    if (moodSize.width >= CGRectGetWidth(self.contentView.bounds) - 2 * kMinSideOffset) {
	        moodWidth = CGRectGetWidth(self.contentView.bounds) - 2 * kMinSideOffset;
	    }else if (moodSize.width <= kMinMoodWidth){
	        moodWidth = kMinMoodWidth;
	    }else{
	        moodWidth = moodSize.width;
	    }

	    self.moodButton.frame = CGRectMake((CGRectGetWidth(self.contentView.bounds) - moodWidth) / 2, CGRectGetHeight(self.contentView.frame) - kWholeHeight, moodWidth, kMoodButtonHeight);

	    NSString *currentCountStr = [self.timeButton titleForState:UIControlStateNormal];
	    CGSize placeSize = [currentCountStr sizeWithFont:self.timeButton.titleLabel.font];
	    if (placeSize.width > kMaxCountWidth) {
	        currentCountStr = [currentCountStr shortIt];
	        [self.timeButton setTitle:currentCountStr forState:UIControlStateNormal];
	        placeSize = [currentCountStr sizeWithFont:self.timeButton.titleLabel.font];
	    }

	    CGSize unitSize = [self.unitLabel.text sizeWithFont:self.unitLabel.font];
	    CGFloat placeWidth = placeSize.width + unitSize.width;
	    CGFloat realPlaceWidth = placeWidth + 2 * kBetweenStarAndPlace + 2 * kStarWidth;

	    self.leftStarIgv.frame = CGRectMake((CGRectGetWidth(self.contentView.bounds) - realPlaceWidth) / 2, CGRectGetMaxY(self.contentView.frame) - kDownOffset, kStarWidth, kStarHeight);
	    self.timeButton.frame = CGRectMake(CGRectGetMaxX(self.leftStarIgv.frame) + kBetweenStarAndPlace, CGRectGetMaxY(self.leftStarIgv.frame) - 4 * kBetweenOffset, placeSize.width, kPlaceButtonHeight);
	    self.unitLabel.frame = CGRectMake(CGRectGetMaxX(self.timeButton.frame) + 2, CGRectGetMinY(self.timeButton.frame) - 3 * kBetweenOffset, unitSize.width, kUnitLabelHeight);
	    self.rightStarIgv.frame = CGRectMake(CGRectGetMaxX(self.unitLabel.frame) + kBetweenStarAndPlace, CGRectGetMinY(self.leftStarIgv.frame), kStarWidth, kStarHeight);
	}

今天还写了个数字加逗号分开的category，很弱，我觉得，大神看到请指正。

如给你

	NSString *count = @"77582587758258"

如何操作输出得

	77,582,587,758,258

我瞎弄了一个（NSString 的 category），至少可以跑，输出没问题。

	- (NSString *)commaNumber:(NSInteger)num
	{
	    NSString *another = self;
	    NSInteger sLength = another.length;
	    NSMutableArray *sArray = [NSMutableArray arrayWithCapacity:5];
	    while (sLength >= 4) {
	        NSString *cut = [another substringWithRange:NSMakeRange(sLength - 3, 3)];
	        another = [another substringToIndex:sLength - 3];
	        [sArray insertObject:cut atIndex:0];
	        sLength = another.length;
	    }
	    for (NSString *s in sArray) {
	        another = [[another stringByAppendingString:@","] stringByAppendingString:s];
	    }
	    return another;
	}
