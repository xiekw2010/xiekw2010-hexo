---
title: node 学习笔记之 stream
date: 2017-04-12 11:52:41
tags:
---

## stream handbook

如果对 stream 还不熟悉，先请阅读下这篇文章 [stream-handbook](https://github.com/substack/stream-handbook)。

总结下这篇文章：

1. stream 使得编程模型概念变简单。
2. stream 可以节省内存开销，从 readable 到 writeable，readable 读多少，writeable 写多少，用完就删除。

stream 的关键就是 ```pipe``` 这个 api 的实现（也就是 linux 里面的 ```|```），这个 api 的核心逻辑如下：

```js
readable.on('data', function (data) {
  if (false === writable.write(data)) {
    readable.pause()
  }
})

writable.on('drain', function () {
  readable.resume()
})
```

```writable``` 是一个可写流```Writable```对象，上游调用其 write 方法将数据写入其中。
writable 内部维护了一个写队列，当这个队列长度达到某个阈值（state.highWaterMark）时，
执行 write() 时返回 false，否则返回 true。

于是上游可以根据 write() 的返回值在流动模式和暂停模式间切换。

当 write() 返回 false 时，调用 readable.pause() 使上游进入暂停模式，不再触发 data 事件。

但是当 writable 将缓存清空时，会触发一个 drain 事件，再调用 readable.resume() 使上游进入流动模式，继续触发 data 事件。

## stream 的类型

```js
var Stream = require('stream')

var Readable = Stream.Readable
var Writable = Stream.Writable
var Duplex = Stream.Duplex
var Transform = Stream.Transform

```

Readable.pipe(Duplex).pipe(Writable)

Readable 可以 pipe 到 Writable 里去，反之不行。

但是 Duplex 即是 Readable 又是 Writable，所以这里可以做为中间的桥接。

Transform 继承自 Duplex，但是与 duplex 的差别在于，pipe 进它 writeable 里的 buffer 经过转换后自动添加到 readable，一般来说，不必重写它的 _read 和 _write 方法，只要实现转换方法 _transform 就行了。

## 怎么实现一个类似 gulp 的 pipe 功能

大致 api 功能：

```js
  return gulp.src(paths.scripts)
    .pipe(sourcemaps.init())
    .pipe(coffee())
    .pipe(uglify())
    .pipe(concat('all.min.js'))
    .pipe(sourcemaps.write())
    .pipe(gulp.dest('build/js'))
```

把 paths.scripts 里的 js 代码经过一系列的 stream 的转换最后输出到 gulp.dest('build/js') 里。

其中 ```sourcemaps.init()```，```coffee()```，```uglify()``` 等等是返回一个个 Duplex 的 stream。

这里大致实现一个最简单的 demo：

``` js
const fs = require('fs')
const thr = require('through2')

const rs = fs.createReadStream('./pipe.js')
const ws = fs.createWriteStream('./sout')

rs
  .pipe(replaceConstWithVar())
  .pipe(toUpperCase())
  .pipe(ws)

function toUpperCase() {
  return thr(function (chunk, enc, next) {
    const str = chunk.toString('utf8').toUpperCase()
    next(null, Buffer.from(str, 'utf8'))
  })
}

function replaceConstWithVar() {
  return thr(function (chunk, enc, next) {
    const str = chunk.toString('utf8').replace(/const/ig, 'var')
    next(null, Buffer.from(str, 'utf8'))
  })
}
```

这里实现了 Transform 的 stream，对进来的数据做一系列的转换。

可以到的结果如下：

```js
VAR FS = REQUIRE('FS')
VAR THR = REQUIRE('THROUGH2')

VAR RS = FS.CREATEREADSTREAM('./PIPE.JS')
VAR WS = FS.CREATEWRITESTREAM('./SOUT')

RS
  .PIPE(REPLACEVARWITHVAR())
  .PIPE(TOUPPERCASE())
  .PIPE(WS)

FUNCTION TOUPPERCASE() {
  RETURN THR(FUNCTION (CHUNK, ENC, NEXT) {
    VAR STR = CHUNK.TOSTRING('UTF8').TOUPPERCASE()
    NEXT(NULL, BUFFER.FROM(STR, 'UTF8'))
  })
}

FUNCTION REPLACEVARWITHVAR() {
  RETURN THR(FUNCTION (CHUNK, ENC, NEXT) {
    VAR STR = CHUNK.TOSTRING('UTF8').REPLACE(/VAR/IG, 'VAR')
    NEXT(NULL, BUFFER.FROM(STR, 'UTF8'))
  })
}
```

## 参考

- [Node.js Stream - 基础篇](http://tech.meituan.com/stream-basics.html)
- [Node.js Stream - 进阶篇](http://fe.meituan.com/stream-internals.html)
- [Node.js Stream - 实战篇](http://fe.meituan.com/stream-in-action.html)
- [streamify-your-node-program](https://github.com/zoubin/streamify-your-node-program)
- [stream-handbook](https://github.com/substack/stream-handbook)