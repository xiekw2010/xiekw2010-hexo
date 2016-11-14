---
title: 可以做 js 面试实战题吗
date: 2016-11-02 15:01:27
tags:
---

第一题：

```js
var obj = {
  "bigKey1": [{
      "_key": "_key",
      "_iOSkey": "_iOSkey",
      "_AndroidKey": "_AndroidKey",
      "_说明": "_说明",
      "_备注": "_备注",
      "_服务端打点": "_服务端打点",
      "_degree": "_degree",
      "_spma": "_spma",
      "_spmb": "_spmb",
      "_spmc": "_spmc",
      "_spmd": "_spmd",
      "_args": "_args",
      "title": "title",
      "url": "url",
      "item_id": "item_id",
      "ju_id": "ju_id",
      "order_id": "order_id",
      "spm": "spm",
      "spm-cnt": "spm-cnt"
    },
    {
      "_key": "JuSplash",
      "_iOSkey": "JHSSplashViewController",
      "_AndroidKey": "",
      "_说明": "聚划算-闪屏",
      "_备注": "",
      "_服务端打点": "",
      "_degree": "",
      "title": "",
      "url": "",
      "item_id": "",
    },
    {
      "_args": "",
      "title": "",
      "url": "",
      "item_id": "",
      "ju_id": "",
      "order_id": "",
      "spm": "",
      "spm-cnt": ""
    },
    {
      "url": "",
      "item_id": "",
      "ju_id": "",
      "order_id": "",
      "spm": "",
      "spm-cnt": ""
    },
    {
      "_key": "JuVoice",
      "_iOSkey": "JHSVoiceView",
      "_AndroidKey": "VoiceRecognizeActivity",
      "_说明": "聚划算-语音识别",
      "_备注": "",
      "_服务端打点": "",
    }]
}

// 有 obj，里面有个 bigKey1 , bigKey1 对应的数组里的第一个对象是『模板』。
// 期待输出，把 bigKey1 对应数组里的其他对象自动填充『模板』对应所有的 key，如果其他对象没有这个 key 的值，那么它的 value 是 ''
```

第二题：
```js
var obj1 = {
  "_key": "JuSplash",
  "_AndroidKey": "hello,yingying",
  "_说明": "xxxx",
  "_备注": "",
  "_服务端打点": "",
  "_degree": "",
  "_spma": "a240c",
  "_spmb": "7662933",
  "_spmc": "0",
  "_spmd": "0",
  "_args": "",
  "__key1": "spm-cnt",
  "__value1": "1",
  "__key2": "ju_id",
  "__value2": "2"
}

// 有 obj1，里面有 __key1，__key2, __value1, __value2，它们是一一对应的关系, __key 和 __value 的个数不是写死的
// 期待输出下面的 obj2，自动把 __key{X} 和 __value{X} 做为 couple 存进原来的 obj1，并且删掉 __key{X}, __value{X}
var obj2 = {
  "_key": "JuSplash",
  "_AndroidKey": "hello,yingying",
  "_说明": "xxxx",
  "_备注": "",
  "_服务端打点": "",
  "_degree": "",
  "_spma": "a240c",
  "_spmb": "7662933",
  "_spmc": "0",
  "_spmd": "0",
  "_args": "",
  "spm-cnt": "1",
  "ju_id": "2"
}
// 提示： 这里的 __key1 有，但__value1 没有，那么不用删除 __key1
// 最佳答案：一次循环就出结果了
```

第三题：
```js
// 用异步的方式递归读出某个文件夹的里所有文件，结果放在一个数组里
function readDirPromise(dir) {
  return new Promise(resolve, reject) {
    fs.readFile(dir, (err, files) => {
      if (err) reject(err)
      resolve(files)
    })
  }
}

function* readDir(dir) {
  const files = yield readDirPromise(dir)
  const res = []
  for (var i = 0; i < files.length; i++) {
    const subDir = files(i)
    if (isDir(subDir)) {
      res.push(yield readDir(subDir))
    } else {
      res.push(subDir)
    }
  }

  return res
}

当然如果要更详细一点，这里应该加一些 try catch 来保证错误的捕获

```
