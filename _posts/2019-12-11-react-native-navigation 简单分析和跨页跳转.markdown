---
layout: post
title:  "react-native-navigation 简单分析和跨页跳转"
date:   2019-12-11 19:36:48 +0800
categories: React Native

---



# react-native-navigation 简单分析和跨页跳转

[APP](http://ju.outofmemory.cn/tag/APP/) [Javascript](http://ju.outofmemory.cn/tag/Javascript/) [Pop](http://ju.outofmemory.cn/tag/Pop/) [React](http://ju.outofmemory.cn/tag/React/) [Navigation](http://ju.outofmemory.cn/tag/Navigation/) [Native](http://ju.outofmemory.cn/tag/Native/)

虽然 [react-native-navigation ](https://github.com/wix/react-native-navigation)是 Facebook React Native 官方文档 [推荐的导航库之一 ](https://facebook.github.io/react-native/docs/navigation.html)，但我也不得不说使用它做 APP 导航主框架的体验简直糟糕透了。当然，这本身可能就是 React Native 自身的问题。

## 1 react-native-navigation 简单分析

使用 react-native-navigation 首先得理解下它的实现。它独立于 RN Component 的 `componentWillMount `/ `componentWillUnmount `接口实现了一套自己的事件机制，最重要的可能是 `willAppear `/`willDisappear `。它提供了一套页面堆栈操作和切换动画， `push `可以将目标页面切换到最上方， `pop `可以返回上一页。

可能是为了性能或者设计使然， `push `的时候不会销毁当前页。也就是说，在 A 页面里 `push `跳转到B 页面，不会 Unmount A 页面的 `Component `。 不过在 B 页面 `pop `回 A 页面时，的确会 Unmount B 页面的 `Component `。这也意味着，整个导航路径是一个页面堆栈，只要在堆栈里页面的 `Component `，都不会被 Unmount。

## 2 页面堆栈的问题

这有时候会导致一些很严重的问题。有些情况下，特定的 `Component `可能会占用唯一的系统资源，比如：麦克风、照相机等。这些 `Component `在实现的时候往往只考虑了 React Native 的接口，在`componentWillUnmount `的时候释放占用的资源。它们不会预料到与 react-native-navigation 的结合，专门提供一个 `willDisappear `时释放资源的接口，而且有些情况下也未必能这样做。

如果 A 页面在使用这些 `Component `已经占用了麦克风或者相机，B 页面也要使用这些 `Component `，那么从 A `push `跳转到 B 时，A 页面的资源不会被释放，B 页面就可能会遇到麦克风不可用，或者相机无法初始化等问题。

解决这个问题，最简单的办法是调整页面交互顺序，保证使用这些独占系统资源的页面永远在堆栈的最顶端，或者使用 Modal Stack，把独占资源的 `Component `放到 Modal 里去 present 然后 dismiss。

## 3 跨页跳转实现

react-native-navigation 只能支持页面堆栈，而且看起来只能支持 push/pop 一个页面，也就是说整个切换过程是串行的，push 顺序是 A->B->A->D ，那么 pop 顺序也只能是 D->A->B->A。

但很可惜地是，在产品经理眼中，是不存在串行页面切换这种限制的。TA 们有时候要求跳转的过程中没 A，但返回的时候要有 A；或者要求跳转的过程中有 A，但返回的时候可以跳过 A，或者甚至直接返回到堆栈最底端。

直接返回栈底很容易，react-native-navigation 提供了 popToRoot 接口，但它没有提供一下子 push 多个页面，或者一下子 pop 多个页面的功能。它也没有类似于 HTML5 的 history API，我们直接对堆栈进行操作，是不太可能的。只能通过它现有的接口想办法。

### 3.1 跨页 push

跳转的过程没有 A，但返回的时候要有 A，这只是一个产品需求。在实现上，是可以变成跳转过程中有 A，但是 A 被快速跳过，返回的时候才会被真正渲染。这样从用户体验上来看，并没有看到 A。代码实现上，可以考虑两种方法：

#### willAppear 结合 didDisappear 做状态控制

在 A 的 `state `里放一个 `isFirstEntry `状态，默认是 `true `。 `willAppear `里判断 `isFirstEntry `则直接跳转到下个页面， `render `里判断 `isFirstEntry `则只渲染一个背景 `View `，否则才渲染正常页面。这样就实现了在页面切换过程中跳过 A。在 的 `didDisappear `里将 `isFirstEntry `置为 `false `。这样在返回的时候 `willAppear `和 `render `表现就和正常返回一样了。

```
  willAppear = () => {
    if (this.state.isFirstEntry) {
      this.props.navigator.push(...);
      return;
    }
    ...
  };
  render() {
    if (this.state.isFirstEntry) {
      // 返回背景 View
    } else {
      // 返回正常 View
    }
  }
  didDisappear = () => {
    this.setState({isFirstEntry: false});
  };
```

#### willAppear 页面计数

在需要更复杂逻辑的地方，可以在 `state `里放一个 `appearTimes `计数器。在 `willAppear `里给计数器加一，这样每次进入页面都会增加计数。通过判断计数器的值，来决定如何 `render `或者跳转。

```
  willAppear = () => {
    this.setState({appearTimes: this.state.appearTimes + 1});
    if (this.state.appearTimes === 1) {
      this.props.navigator.push(...);
      return;
    }
    ...
  };
  render() {
    if (this.state.appearTimes === 1) {
      // 返回背景 View
    } else {
      // 返回正常 View
    }
  }
```

### 3.2 跨页 pop

跳转的过程中有 A，但返回的时候要跳过 A，相当于可以自己操作 pop 的步长。很遗憾，react-native-navigation 没有提供这样的接口。不过我们可以采用一个 trick 的手段，来实现这个逻辑。

假设从 Root->A->B，在 A 的 `state `里放一个 `relayPop `，默认是 `false `。 在 A 跳转到 B 时，通过 `props `传入一个回调： `setParentRelayPop `，B 可以通过这个回调修改 A 的 state `relayPop `为`true `； 在 A 的 `willAppear `中，首先判断 `relayPop `是否为真，如果是真的话，代表是从 B 返回且 B 要求接力返回，那么 A 就直接 `pop `返回到 A 的上级。 B 在返回时，首先通过回调设置 `relayPop`为 `true `，然后再调用 `pop `接口，就实现了跨页返回。

```javascript
// Screen A
  willAppear = () => {
    if (this.state.relayPop) {
      this.props.navigator.pop();  // 接力返回
      return;
    }
    ...
  };
  ...
    // 跳转逻辑某处
    this.props.navigator.push({..., passProps: {
                                  setParentRelayPop: () => this.setState({relayPop: true}) 
                                }});
// Screen B
    // 返回逻辑某处
    this.props.setParentRelayPop();
    this.props.navigator.pop();
```

