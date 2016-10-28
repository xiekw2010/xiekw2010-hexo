---
layout: post
keywords: blog
description: blog
title: "AnchorPoint,Position,Frame"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2014-11-20 17:53:30
---

`AnchorPoint` layer做动画的支点，transform的scale, rotate等从那个点开始。

`Position` layer的anchorPoint在superLayer坐标系统中的位置。

问题一:

layer.anchorPoint 改变了，为啥它的view的frame也变了？

答: layer.position并没有变化。layer要适应position就是anchorPoint在superLayer中的位置不变，只要改变其自己的originPoint的位置。

eg1:

	view.frame = (CGRect){(CGPoint){200, 100}, (CGSize){100, 100}}
	then
		view.position = {250, 150} //default
		view.anchorPoint = {0.5, 0.5} //default

	if set
		view.anchorPoint = {1.0, 1.0}
	then
		view.position = {250, 150} //not change
		view.frame = {150, 50}; //to adjust the new anchorPoint

	Reset view's frame, anchorPoint.
	if set
		view.position = {150, 50};
	then
		view.anchorPoint = {0.5, 0.5};
		view.frame = (CGRect){(CGPoint){100, 0}, (CGSize){100, 100}} // to adjust the new position

问题二:

如何避免，改了anchorPoint, view的frame不跟着变?

答: setLayer.position to adjust it.（为啥不设frame, 设置frame会牵扯到layer.transform。）

eg2:

	view.frame = (CGRect){(CGPoint){200, 100}, (CGSize){100, 100}}
	then
		view.position = {250, 150} //default
		view.anchorPoint = {0.5, 0.5} //default

	if set
		view.anchorPoint = {1.0, 1.0}
		view.position = {300, 200}
	then
		view.frame = (CGRect){(CGPoint){200, 100}, (CGSize){100, 100}} //still the origin, but the anchor point changed

结论:

如果set new anchorPoint, 那么 position也得重新考虑了。
如果想要frame不变化，那么有公式:

`#define DX_ChangeLayerAnchorPointAndAjustPositionToStayFrame(layer, nowAnchorPoint) \
CGPoint DX_lastAnchor = layer.anchorPoint;\
layer.anchorPoint = nowAnchorPoint;\
layer.position = CGPointMake(layer.position.x+(nowAnchorPoint.x-DX_lastAnchor.x)*layer.bounds.size.width, layer.position.y+(nowAnchorPoint.y-DX_lastAnchor.y)*layer.bounds.size.height);\`


这个宏改变了layer.anchorPoint后又马上跟着改变了它的posistion，效果是让他的frame保持不变。
