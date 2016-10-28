---
title: Instalocation 2.0
date: 2013-08-19 17:53:30
---

经历了半个多月，今天终于把2.0做完了。

首先让我吐槽一下这个2.0来的太快，当我上个月月底还在为1.3的一些界面问题，微博微信分享问题头大时候，领导就把GUI，team leader叫到会议是说，“我们下午就来个头脑风暴，想想2.0要加啥东西吧！”

最后商量下来，需求还不是很过分，就加个滤镜，景深，和交叉推广。还有几个新主题。一个个说吧：

1. 滤镜

	看了点苹果自带CoreImage吧，里面有个控制颜色管道的东西，挺好的，但是总的来说，滤镜还要一个个去调，这有点拖时间。

	想想，既然公司里有人做过GPUImage滤镜，那就索性用他的好了，何必重复造轮子呢。我之前有篇博客讲过，异步处理的滤镜图片的。到[之前的博客](http://xiekw2010.github.io/2013/07/setimagewithurl:placeholder:/)。

	是不是有点不足？就是你tableview来回滚动的时候，它的cell是会来回重新加载滤镜的，也就是说少了一个缓存机制。这很关键。

	解决方法，对UIImageView搞一个共享的NSCache，暂且叫他为FXImageCache（subclass NSCache）,当一个图片需要加滤镜的时候，每个滤镜名对应一个key，每个滤镜后的图片对应一个value。(NSCache有点像一个dictionary，嗯哼？)，嗯，当需要某个原图需要某个滤镜名的图片时，我们是先到cache里去找，找到就用，找不到再去生成，这样就能解决在tableview滚动的时候重复添加滤镜的问题了吧。考虑到全局，当另一个图片需要加载的时候，那就把上一个图片的cache全部清除掉，生成属于当前这个图片的cache。

	好了，现在问题是如何在下一个图片需要滤镜的时候，把上一个的cache清除光。

	- 笨办法，就是自己到程序里去找生成新图片的地方。“哦~我这里是生成了新图片，那我就把上一次的删了~”。这样做就是有点不太稳，万一哪个地方也是生成新图片，而你忘记了去删除cache，这样你的实时滤镜显示的图片可能就是上次的那一批，效果很差，丢脸。这里就不谈cache有多大，删除同步还是异步的问题了。

	- 机智一点的办法，对FXImageCache设置一个imageHash的int属性。
	在用setImageWithFilterName:(NSString *)name placeHolder:(UIImage *)placeholder函数里用这个属性与placeholder的hash比较，如果是相同的，那就说明还是同一张图片。不需要清除之前的cache。如果是不相同的话，那就要把之前的cache清除掉了，在把当前的FXimageCache的imagehash置为placeholder的hash。

	这个源码我还是open吧。猛击[这里](https://github.com/xiekw2010/UIImageView-Filter)

2. 景深

	这东西不难，有个模板的叫DLCImagePickerController的repo可以研究下。

3. 交叉推广

	这个蛋疼的东西涉及商业机密，这里就不多说了。把公司里的很多人搞的晕晕乎乎。
