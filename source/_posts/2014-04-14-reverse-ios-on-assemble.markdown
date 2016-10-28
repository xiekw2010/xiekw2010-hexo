---
layout: post
keywords: blog
description: blog
title: "Reverse iOS on assemble"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2014-04-14 17:53:30
---

### 前提

1. 安装了各种软件，软件及软件安装说明见[此](http://http://xiekw2010.github.io/2014/03/26/iossteps/)

2. clutch你感兴趣的程序，class-dump到你感兴趣的方法，IDA定位到你感兴趣的方法。

### 步奏

1. 定制你的debugserver([实现细节](http://iosre.com/forum.php?mod=viewthread&tid=52&highlight=lldb)), 我按照教程做的[现成下载](http://pan.baidu.com/s/1sjFE55b)(打开后，只用到debugserver), 把它放到*iOS*设备的/usr/bin目录下，再cd到此目录授权, 输入'chmod 777 debugserver'

2. ssh到iOS设备，输入'debugserver *:1234 -a "Instagram"' (iOS设备启动某程序的debug服务)

3. 在os X终端里, 输入'lldb', 再输入'platform select remote-ios' (开启lldb并选择远程iOS调试模式)

4. process connect connect://10.10.0.170:1234 (连接到已经开启的debug服务的设备，ip是iOS设备ip，端口固定1234)

5. 输入'c'(继续Instagram的进程，不然你不觉得界面卡住了吗), 继续输入'image list -o -f'找到"Instagram"的ASLR，这里假设是0xaa000。

6. 找到IDA里你感兴趣的代码地址，如0x1c9e44，这个是这段代码的静态地址，而该代码真正的运行地址还得加上一个偏移量，所以我们的断点地址应该是，real = 0x1c9e44 + 0xaa000。 设置断点'br s -a 0x273e44'

7. 玩弄Instagram, 让程序break在你设的地方。打印出IDA里寄存器的值, 'po (char*) $r0'



##### End

感谢周大哥对我最后几步的指点。
