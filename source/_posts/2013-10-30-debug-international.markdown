---
title: Debug international
date: 2013-10-30 17:53:30
---

每个app都有一个target，每个target都有一个scheme。

这个scheme可以定义一些环境变量来使你的调试进行的更为轻松。比较著名的环境变量有"-NSZombieEnabled"。

这里主要针对的是多语言环境调试。有两个argument很有用。

针对Argument passed on launch:

"-NSDoubleLocalizedStrings YES"  每行NSLocalizedstring会double一下。这针对日语这种蛋疼语言调试很有效果，当你的日语翻译还没有给你翻译完的时候。

"-NSShowNonLocalizedStrings YES"  可以检查出app里没有被Localized的string。

"-AppleLanguages (es)" 把app的语言改为括号内的语言。注意，不论你的设备是什么语言环境，只要设了这个argument，那么在你设备上跑的app的程序语言就是括号内设置的语言了。

一些常用括号内语言缩写

+ zh_CN 中文 (中国)
+ zh_HK 中文 (香港)
+ zh_TW 中文 (台湾)
+ en 英文
+ ja 日语
+ de 德语
+ es 西班牙语
+ fr 法语
+ ru 俄语
+ it 意大利
