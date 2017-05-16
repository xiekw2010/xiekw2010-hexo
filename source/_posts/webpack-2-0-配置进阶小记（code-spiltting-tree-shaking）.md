---
title: webpack 2.0 配置进阶小记（code-spiltting & tree-shaking）
date: 2017-05-16 14:35:47
tags:
---

## code-splitting

webpack 一个亮点功能就是 code-splitting 了。

code-splitting 一般应用在以下两种情况：

1. 分割 app 的 vendor 代码。这样，当 app 改动业务代码的时候，vendor 代码不受到改变，那么就可以有效利用游览器对之前 vendor 代码的缓存。
2. 对有 [SPA](https://en.wikipedia.org/wiki/Single-page_application) 有些不常用的页面做按需加载，用 webpack.config 里的 publicPath 对这些 js 代码做部署，然后当需要用到时再通过 jsonp 的方式加载到 html 里。


### 配置怎么写呢？

#### 1. vendor 缓存的配置：

```js
var webpack = require("webpack");

module.exports = {
  entry: {
    app: "./app.js",
    vendor: ['moment'] // 项目 node_modules 下的一些依赖
  },
  output: {
    path: __dirname + '/dist',
    
    // 注意是『chunkhash』而不是『hash』！
    // 详情见：https://webpack.js.org/guides/caching/#generating-unique-hashes-for-each-file
    filename: "[name].[chunkhash].js", 
    publicPath: __dirname + '/dist/'
  },
  plugins: [
 	// 为啥需要 manifest，当业务代码改动时，vendor 的 hash 不会受影响
    new webpack.optimize.CommonsChunkPlugin({
      name: ['vendor', 'manifest'] 
    }),
  ]
};
```

#### 2. 按需加载的配置

```import()``` VS ```require.ensure()```

- ```require.ensure()``` 是 webpack 对按需加载的语法标记，既如果用了这种语法，webpack 会你的 js 做静态分析然后对```require.ensure()``` callback 里的代码做 jsonp 的按需加载封装。

- ```import()``` 是 webpack 2.0 新引入对```require.ensure()``` promise 化的『语法糖』（可能实现机制有些不同，没有细看源码）

看完官方 [guide](https://webpack.js.org/guides/code-splitting-async/)，我个人可能更偏向于使用```require.ensure()```，原因如下：

1. IDE 不报错
2. 可以觉得哪些按需加载的代码执行与否（啥意思呢？我可能按需加载了 a.js 代码，但是我可以并不去执行这段 js 代码，控制力度更细一点）

另外官方文档里也使用了 ```async await``` 的语法，但是用了这种语法糖后，生成最后的代码大了很多（把 generator 的运行时也打包进去了）

##### publicPath 的用处

在还没用到 code-splitting 之前，一直没认识到 output 的 publicPath 的作用。

publicPath 的作用：

有些页面需要按需加载的资源地址。（这个地址如果放在本地 static server 里，那可能就是 __dirname + '/somepath/', 如果放在 cdn，那可能就是 https://somehost.com/somepath/）

放到 code-splitting 场景就是，按需加载时（jsonp 请求 js 资源时） js 的资源地址。

## tree shaking

##### CMD VS ES2015 modules（以下简称 modules）

CMD 和 modules 的区别简单来说就是，运行时加载与编译时加载，看个例子：

```js
// CMD
let { stat, exists, readFile } = require('fs');

```

这段代码是在运行时把整个 fs 都加载了，虽然语法看上去像只加载了这三个属性。

```js
// ES6模块
import { stat, exists, readFile } from 'fs';
```

这段代码是只加载了 fs 的三个属性，这种加载称为编译时加载。

tree shaking 做的事情就是把 modules 里没有被 import 过的对象最终不打包进 bundle.js 里。

webpack 2.0 默认支持 modules 的加载方式。

***但是***，现在的前端都是用 es2015 的语法来写的，webpack 默认对这些语法是不支持的，所以还是要引入 babel-loader 来进行转换。

比如：

```js
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
        options: {
          presets: ['react', 'es2015',],
        },
        exclude: /node_modules/,
      }
    ]
  },
```

上面这段代码开启了对 react，es2015 语法的支持。

***但是***，一但使用了 babel-loader 的 es2015，webpack 2.0 默认的 modules 就会被 babel 转成 commonjs 的模块了，享受不到 tree shaking 的福利了。

所以现在一般如果用 babel-loader 的 es2015，都会加一个 ```{ modules: false }```，上面这段配置改动后如下：

```js
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
        options: {
          presets: ['react', ['es2015', { modules: false }],],
        },
        exclude: /node_modules/,
      }
    ]
  },
```

这样就告诉 babel-loader，不要把 modules 转成 CMD 了。
