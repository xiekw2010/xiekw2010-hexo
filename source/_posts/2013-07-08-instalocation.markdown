---
title: InstaLocation
date: 2013-07-08 17:53:30
---

InstaLocation, InstaPlace, InstaFood, InstaWeather, 这世界上有好多insta。

希望我的Instalocation也能占有一席之地，虽然说我觉得它挺无脑操作的。希望大家能像喜欢大胸妹一样地喜欢我这个App。

总结一下吧，我要做的就是：

1. 用AVFoundation写一个照相机，这个照相机要有拍照，切换镜头，切换闪光灯的功能。
	+ 第一件事就是要搞定两个toolbar，这两个toolbar间是有关系的，比如说下面那个toolbar有个拍照按钮，按了拍照按钮后
	  要把上面toolbar的闪光灯和切换去掉，于此同时自己也要变成分享界面的toolbar按钮了，在分享界面按了撤掉后，又要把上面的两个相机功能按钮给打开了。

	+ 拍照功能，这是一个async方法，从拍照到生成图片中间是有一段时间过程的，这段过程如果对AVpreviewLayer不加以遮盖的话，那就会感觉这个相机反应超慢，用户体验会不好。所以大多数的相机应用都会做一个拍照的动画效果来遮盖掉这层previewlayer。

	+ 切换镜头和闪关灯官方提供相关操作代码。这里还是觉得了解它们之间的逻辑比较重要，avsession，avdevice，deviceoutput, deviceinput, deviceConnection之间的关系吧（这是[官方文档](http://developer.apple.com/library/ios/#documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html#//apple_ref/doc/uid/TP40010188-CH5-SW2)）

	+ 补充说明，如果想做一个有点击定焦功能的相机的话，首先要加一个tapgesture，然后要计算出gesture的点在previewlayer的preview里的坐标（难点）。官方示例代码（AVCam）中有convertToPointOfInterestFromViewCoordinates演示.

2. 写一个模版爸爸，这个爸爸支持各种改变图标，文字，时间，操作。然后对43个不同模版，造43个儿子，体力活。
	+ 每个儿子都有尺寸限制，每个儿子的线条，文字，图标之间是互相限制的。也就是说，layoutsubviews会写到接近崩溃。

	+ 各种弹出的编辑界面是不一样的。

3. 界面要好看，就必须要知道怎么贴图，这个商店图贴起来真闹心啊。

4. 商业模式比较复杂，有两种锁，一种是分享锁，还有种是购买锁。不同锁会有不同的界面变化。还有种添加水印，有首歌歌词写的真好，“56个名族，56朵花”. 我这是43个儿子，43个水印位置。

5. iPad的挑战在哪里？
	+ 支持旋转，编辑界面怎么支持旋转是一个问题。我预见到估计要把所有的view模式改为viewcontroller模式。
