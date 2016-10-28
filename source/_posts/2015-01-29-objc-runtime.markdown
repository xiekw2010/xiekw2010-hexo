---
layout: post
keywords: blog
description: blog
title: "Objc runtime"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-01-29 17:53:30
---

这里我只是做一个笔记，[原文](http://tech.glowing.com/cn/objective-c-runtime/) 解释的很详细，推荐一下！

Objc 核心是msg_send。

当使用objc_msgSend(obj, foo)时候。做以下几步:

1. 通过obj的isa指针定位到它的class;
2. 在class的method里找foo(class_method_list还有一个objc_cache, 先在cache里找, 如果找不到再到list里去遍历)
3. class中如果没有foo, 去superclass里找。
4. 找到后就去执行它的imp

如果找不到，程序抛出unrecognized selector sent to前，有三次机会拯救:

1. Method resolution(+resolveInstanceMethod: or +resolveClassMethod:)
2. Fast forwarding(-forwardingTargetForSelector:), 这里把锅传给其他对象。
3. Normal forwarding(-methodSignatureForSelector: if exists then -forwardInvocation:)
