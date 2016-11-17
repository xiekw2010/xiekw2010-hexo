---
title: Learn Parse Server
date: 2016-11-15 17:47:17
tags:
---

最近想把 http://bonwechat.com 后端的数据服务改造成 Parse Server 的。

## 什么是 Parse Server?

可以理解为是一个数据服务，对比国内的 leancloud，但它是开源，可以进行很多定制化的事情。马上就要有个自己的云服务了，这件事情，想想就令人激动！

## 为什么是 Parse Server?

1. 成熟的客户端数据 SDK 接口 --- iOS，android，php，js，windows等等的（虽然现在只用到 js，- -!）。
2. 有个美观强大 dashboard 界面。
3. 一整套的解决服务端恼人的 social，ACL，用户，权限等的数据关系。
4. 易于在 js 系统中部署。
5. 移动设备 notification push 机制。
5. cloud code 可以进行定制定时服务在云端跑。
7. 可伸缩扩展的服务端和数据库架构。
6. 日志系统，虽然这是 web 框架的标配。

感觉有点像哥伦布发现新大陆了，好兴奋哦！

## 我是怎么迁移原来的 [egg](https://eggjs.org/) 服务的？

有两个目的：

1. 保持原来的网站和接口服务
2. 把原来的数据服务用 parse server 来存储

第一个目的：

  要做到其实很简单，保持原来的 mongo 和 egg 服务

第二个目的：

  有三件事情：

  1. 把 parse server 和 parse dashboard 要结合起来，并有一个专门服务于这个 parse 后台的 mongo
  2. 把原来的 egg mongo 的数据导入到新的 mongo parse 后台里去。
  3. egg 的存数据库服务现在改成 egg 做为 parse 客户端调用 parse 后台的服务来存数据了。

  目标1：

    另起一个 express app，把 parse server 和 parse dashboard 作为 middleware 加进去。f8app 就是这样做的。后续还可以加入 graphql

  目的2：

    写一个导入的脚本就行了，用 monk 从原来的数据库取数据，再存入到新的 parse 后台去

  目的3：

    把 egg 里原来用 monk 存的代码改成 parse 存就行了。

  so easy！  

## 一些 parse SDK 的小技巧

1. 原来为 egg 专门做了数据的分页加载功能，现在用 parse SDK 可以轻松实现

```js
const getTitle = r => r.get('desc')
const step = 100

const Look = Parse.Object.extend('Look')
const query = new Parse.Query(Look)
query.descending("crawledAt")
query.limit(step)

let skip = 0
let sentil = [1]
let res = []
while (sentil.length > 0) {
  query.skip(skip)
  sentil = (yield query.find()).map(getTitle)
  res = res.concat(sentil)
  console.log('skip is', skip)
  skip += step
}

console.log('res is', res.filter(r => !!r.length).length)

return res
```  

2. 字符串匹配查询，这里因为要模糊匹配，还是用正则最稳，虽然有点牺牲效率，没有用到数据库 index 的功能

```js
const target = 'parse server 强无敌'

const Look = Parse.Object.extend('Look')
const tquery = new Parse.Query(Look)
tquery.descending("crawledAt")
tquery.matches('title', new RegExp(`.*${target}.*`))

const dquery = new Parse.Query(Look)
dquery.descending("crawledAt")
dquery.matches('desc', new RegExp(`.*${target}.*`))

const res =  yield Parse.Query.or(tquery, dquery).find()
return res.map(r => [r.get('title'), r.get('desc')])
```

## 总结一下

有了 parse server 后，省去了好多后台数据的开发工作，并且同时有了各种平台的数据服务 SDK 了，它的标准已经实现了覆盖 90% 的数据服务。剩下的 10% 也完全可以由灵活的 cloud code cover 到。

稍微有些蛋疼的就是，每次新建个 app 要重新发布一下，必须得在后台手动加入，parse 并没有开源增加 app 的总 dashboard。

不过现在的 dashboard 也够用了。
