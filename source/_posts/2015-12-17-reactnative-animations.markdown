---
layout: post
keywords: blog
description: blog
title: "ReactNative Animations"
categories: [Archive]
tags: [Archive]
group: archive
icon: file-alt
date: 2015-12-17 17:53:30
---

ReactNative不仅能做到H5那样的页面元素布局，还能做到像Native一样自然的动画。目前它有两套动画系统---`Animated`和`LayoutAnimation`。`Animated`更注重小而精细的动画控制，`LayoutAnimation`更关注全局布局类型的动画。

##Animated

先来看个简单的例子，在UIKit中做个简单的动画：

```ruby
  [UIView animateWithDuration:0.3 animations:^{
    aView.transform = CGAffineTransformMakeScale(1.2, 1.2);
  } completion:^(BOOL finished) {
  }];
```

在RN中怎么做这个动画呢？

```js
class Playground extends React.Component {
  constructor(props: any) {
    super(props);
    this.state = {
      bounceValue: new Animated.Value(0),
    };
  }
  render(): ReactElement {
    console.log('rending...');
    return (
      <Animated.Image                         // Base: Image, Text, View
        source=【【uri: 'http://i.imgur.com/XMKOH81.jpg'】】
        style=【【
          flex: 1,
          transform: [                        // `transform` is an ordered array
            {scale: this.state.bounceValue},  // Map `bounceValue` to `scale`
          ]
        】】
      />
    );
  }
  componentDidMount() {
    this.state.bounceValue.setValue(0.8);     // Start large
    Animated.timing(                          // Base: spring, decay, timing

      this.state.bounceValue,                 // Animate `bounceValue`
      {
        toValue: 1.2,                         // Animate to smaller size
        duration: 2000,
        delay: 1000,
      }
    ).start();                                // Start the animation
  }
}

```

这段JS代码，有三点需要解释一下：

1. 为什么`render`方法这里只执行一次，尽管这个`Animated.Image`的`transform`的值在做变化
2. `Animated.Image`这是个什么组件？它和`Image`有啥关系呢？
3. `bounceValue: new Animated.Value(0)` 的`Animated.Value`是什么?
4. 最后用到了native的Core Animation吗？怎么桥接的？

解释：

1. 这里要回答第一个问题，需要先解释下第二个问题。什么是`Animated.Image`?

	看Animated.js里的最后`module.exports`。

![2](http://img4.tbcdn.cn/L1/461/1/ae2faa694da012b1ae2e15b661de865aab4e04fd.png)

	这个Component是通过`createAnimatedComponent`包装的，那么这层包装做了什么呢？
	对传给Component的每个含有Animated属性的值做监听。下次`requestAnimationFrame`的时候，就去检测这些值有没有被修改，如果修改了，就用这个组件的setNativeProps来更新UI，这样做是不会重新渲染这个组件的整个DOM tree的，[详情说明](http://facebook.github.io/react-native/docs/direct-manipulation.html#content)。

2. 回答第一个问题，`transfrom`的值变化是内部组件由`requestAnimationFrame`而导致的，而不是传统的改变`state`或者`props`来驱动的。
3. 这个`Animated.Value(0)`是与`Animated.Component`约定好的一个属性类，当这个值按照动画曲线变化时，`Animated.Component`会追踪这个值，并更新自己那些用到`Animated.Value`的UI 属性，从而达到动画的效果。
4. RN Animated系统没有用到native的动画系统。是自己用JS实现了一套动画系统，可能为了平台兼容性。

总结: 整个过程如图所示

![3](http://img3.tbcdn.cn/L1/461/1/22fdbf5e9a519fcfcdbcd715d429fffd221d911e.png)

##LayoutAnimation


先看看用法：

```js
var App = React.createClass({
  componentWillMount() {
    // Animate creation
    LayoutAnimation.spring();
  },

  getInitialState() {
    return { w: 100, h: 100 }
  },

  _onPress() {
    // Animate the update
    LayoutAnimation.spring();
    this.setState({w: this.state.w + 15, h: this.state.h + 15})
  },

  render: function() {
    return (
      <View style={styles.container}>
        <View style={[styles.box, {width: this.state.w, height: this.state.h}]} />
        <TouchableOpacity onPress={this._onPress}>
          <View style={styles.button}>
            <Text style={styles.buttonText}>Press me!</Text>
          </View>
        </TouchableOpacity>
      </View>
    );
  }
});
```

可以看到`LayoutAnimation`针对的是某个React的component，当component重新渲染时，它会用到这个animation。

那么它是如何实现的呢？

源码中可以看到它和Native bridge过来的`RCTUIManager`里用到。在看`RCTUIManager`对`LayoutAnimation`的时候，它是封装系统的Core Animation API.

![4](http://img3.tbcdn.cn/L1/461/1/55c206e7041d20ce86bd6b85109785a3242e2351.png)

总结：`LayoutAnimation`是基于Native的animation，只是对单个component渲染到屏幕或者重新渲染时用到，但它的可控的细粒度没有`Animated`高。
