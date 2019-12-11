# 【译】React 优化：虚拟 DOM 详解

作者：Alexey Ivanov、 Andy Barnov | 译：大辉 
原文地址：https://evilmartians.com/chronicles/optimizing-react-virtual-dom-explained

本文将带你学习 React 的虚拟 DOM，并应用这些知识来加速你的应用程序。在这个对框架内部进行全面友好的初步介绍中，我们将揭开JSX的神秘面纱，向您展示React如何做出渲染决策，解释如何找到瓶颈，并分享一些提示以避免常见错误。

React 持续领跑前端世界并且没有下降迹象的原因之一，就是其平易近人的学习曲线。在用 JSX 和 “State vs Props” 的概念武装你的大脑之后，你就可以开始开发了。

但要真正掌握 React，你需要有 React 思想。本文试着从这个方面来帮助你。下面通过我们其中的一个的项目 React table 来了解一下：

> 如果你已经熟悉 React 的工作方式，可以直接跳至“优化：挂载/卸载”这节来继续阅读。

![-w550](http://img20.360buyimg.com/uba/jfs/t21382/23/680430243/113949/90997c5d/5b149c34N982afb9f.png)

（图中是一个在 eBay 的业务中拥有庞大数据的 React table）

通过这上百行动态的并且可过滤的数据行，来理解框架的细节对于保证用户的流畅体验是至关重要。

------

**当程序出错的时候，你很容易感受到，输入文字变得卡顿，复选框检查都需要一秒钟的时间，模态框也很难显示出来。**

------

为了解决这些问题，我们这篇文章要涵盖 React 组件从定义到在页面上呈现（然后更新）的整个生命周期。系好安全带，我们要发车了。

## JSX 的原理

> 在前端圈中称为“转译”的过程，其实用“汇编”来描述是更正确的术语。

React 开发者推荐你使用一种混合了 HTML 和 JavaScript 的语法，即 JSX 来编写你的组件。但浏览器对 JSX 及其语法并不理解。浏览器只理解 JavaScript，所以 JSX 必须转换为 JavaScript。下面是 JSX 的代码，一个有 class 和一些内容的 div。

```html
<div className='cn'>
   Content!
</div>
```

上面的代码转换成常见的 JavaScript 代码只需要调用一个函数传递一些参数即可，如下：

```react
// createElement 接收三个参数 'div'、{className: 'cn'}、'Content!'
React.createElement(
   'div',
   { className: 'cn' },
   'Content!'
);
```

让我们仔细看一下这三个参数：

- 第一个是 *元素的类型* 。对应 HTML 标签名称；
- 第二个是带有 *所有元素属性* 的对象。如果他们没有属性也可以是个空对象；
- 余下的参数是 *元素的子节点* 。元素中的文本也算一个 child，所以一个字符串 ‘Content！’被放置在函数的第三个参数。

你可以想象当我们有很多子节点的时候会发生什么：

```jsx
<div className='cn'>
  Content 1!
  <br />
  Content 2!
</div>
React.createElement(
  'div',                     // 标签名
  { className: 'cn' },       // 元素属性对象
  'Content 1!',              // 1st 子节点
  React.createElement('br'), // 2nd 子节点
  'Content 2!'               // 3rd 子节点
)
```

上面这个函数有五个参数：

- 第一个：元素类型；
- 第二个：属性对象；
- 还有三个子节点：因为其中的一个子节点（ `<br/> `）也是 React 已知的 HTML 标签，所以它也会被描述为一个函数调用。

> 译者注：第三个参数到第五个参数所有的子节点，如果遇到字符串便按照子节点处理，如果碰到标签，需要重复执行上述整个步骤，标签被描述为函数调用。

到目前为止，我们介绍了两种类型的子节点：一种是纯字符串 String，剩下的是可以调用 React.createElement 函数的。然而其他值也可以作为参数：

1. 基本数据类型 `false,null,undefined和true`
2. 数组 `Arrays`
3. React 组件

使用数组是因为可以将子节点分组通过一个参数传递：

```jsx
React.createElement(
  'div',
  { className: 'cn' },
  ['Content 1!', React.createElement('br'), 'Content 2!']
)
```

> 译者注：依然是上面的五个参数的例子这次简化成了三个参数，我们将上一个例子中的后三个参数，放在了一个数组里传递。

当然，React 的厉害之处不是来自 HTML 规范中描述的标签，而是来自用户创建的组件，例如：

```jsx
function Table({ rows }) {
  return (
    <table>
      {rows.map(row => (
        <tr key={row.id}>
          <td>{row.title}</td>
        </tr>
      ))}
    </table>
  );
}
```

组件允许我们将模版分解成多个可重用的块。在示例 [“函数式” ](https://reactjs.org/docs/components-and-props.html#functional-and-class-components)[1]组件中我们接收一个包含表格行数据的对象数组，并返回调用 `React.createElement `的 table 元素，rows 则作为其子节点传入。

无论何时我们都可以像下面这样声明我们的组件：

```
<Table rows={ rows } />
```

在浏览器看来，我们写的其实是这样的：

```
React.createElement(Table, { rows: rows });
```

注意，这次第一个参数不再是描述 HTML 元素的字符串，而是一个在编写组件时定义的 *函数的引用* 。 **函数的属性就是 props**

## 使用组件拼装页面

所以，我们已经将所有 JSX 组件转换为纯 JavaScript，现在我们得到了一堆函数调用，它的参数会被其他函数调用的，或者还有更多的其他函数调用这些参数…（说白了就是函数套函数） 那么它到底是如何转换成组成网页的 DOM 元素的呢？

为此，我们有一个 ReactDOM 库及其它的 render 方法：

```jsx
function Table({ rows }) { /* ... */ } // 声明一个组件

// 渲染一个组件
ReactDOM.render(
  React.createElement(Table, { rows: rows }), // “创建”一个组件
  document.getElementById('#root')            // 将它插入页面
);
```

当 ReactDom.render 被调用，React.createElement 最终也被调用，它返回下列对象：

```
// 这里有一些属性，并且他们对我们很重要。
{
  type: Table,
  props: {
    rows: rows
  },
  // ...
}
```

------

**这些对象在 React 看来便构成了虚拟 DOM。**

------

他们将在所有进一步的渲染中相互比较，并最终转化为真正的DOM（而不是虚拟）。

下面是另一个例子：这次有一个具有 class 属性和几个子节点的 div：

```jsx
React.createElement(
  'div',
  { className: 'cn' },
  'Content 1!',
  'Content 2!',
);
```

变成：

```jsx
{
  type: 'div',
  props: {
    className: 'cn',
    children: [
      'Content 1!',
      'Content 2!'
    ]
  }
}
```

请注意：上面 JSX 原理里说过的在将 JSX 转译成纯 JavaScript 的过程中，我会传递很多参数，第一个参数为元素类型，第二个参数为属性对象，其余的参数我们可以独立传递给 React.createElement 也可以打包以数组的形式传递，最后他们都会在 props 中以 children 为 key 的属性中找到他们。所以无论 children 是作为数组还是参数列表传递都无关紧要 – 在生成的虚拟 DOM 对象中，它们总会被打包在一起。

更重要的是，我们可以直接在 JSX 代码中将 children 添加到 props 中，效果是一样的。比如下面：

```
<div className='cn' children={['Content 1!', 'Content 2!']} />
```

虚拟 DOM 对象构建完成后，ReactDOM.render 会尝试将其转译为浏览器根据以下规则可以展示的 DOM 节点：

- 如果 type 包含一个带有 *String* 类型的标签名称 —— 创建一个标签并附带 props 下所有的属性。
- 如果 type 是一个函数或者类，调用它，并对结果递归地重复这个过程。
- 如果 props 下有 children 属性 —— 在父节点下，针对每个 child 重复以上过程。

最后的结果，我们就得到了如下的 HTML（我们的 table 示例）：

```html
<table>
  <tr>
    <td>Title</td>
  </tr>
  ...
</table>
```

## 重新构建 DOM（Rebuilding the DOM）

> 在实际应用场景，render 通常在根节点调用一次，后续的更新会由 state 来控制和触发调用。

注意标题里的的“重新” (“re”)! 当我们想 *更新一个页面* 而不是全部替换时，React 中真正的魔法就开始了。有很多方法可以实现这种效果。我们先来一种简单的——在相同的节点再次调用 ReactDOM.render。

```jsx
// 第二次调用
ReactDOM.render(
  React.createElement(Table, { rows: rows }),
  document.getElementById('#root')
);
```

这次上面的代码将会与之前看到的有所不同，它不是从头开始创建所有 DOM 节点并将它们放在页面上，而是 React 会启动 [`reconciliation `](https://reactjs.org/docs/reconciliation.html)[2]（或“diffing”）算法，以确定节点树的哪些部分必须更新，哪些可以保持不变。

那么，它是如何工作的呢？其实只有少数几个简单的场景，理解它们将对我们的优化有很大帮助。请记住，现在我们在看的，是在 React Virtual DOM 里面用来代表节点的对象。

- **场景1： type 是一个字符串， type 在调用过程中保持不变， props 也没有改变。**

```jsx
// 更新之前
{ type: 'div', props: { className: 'cn' } }

// 更新之后
{ type: 'div', props: { className: 'cn' } }
```

这是一个简单的示例：DOM 保持不变。

- **场景2： type 仍然是那个字符串， props 不同。**

```jsx
// 更新之前
{ type: 'div', props: { className: 'cn' } }

// 更新之后
{ type: 'div', props: { className: 'cnn' } }
```

由于我们的类型仍然代表 HTML 元素，因此 React 知道如何通过标准的 DOM API 调用来更改其属性，而无需从 DOM 树中删除节点。

- **场景3： type 改变成一个不同的字符串，或者将字符串改成一个组件。**

```jsx
// 更新之前
{ type: 'div', props: { className: 'cn' } }

// 更新之后
{ type: 'span', props: { className: 'cn' } }
```

由于 React 现在认为类型不同，它甚至不会尝试更新我们的节点：old 元素将与其所有子节点一起被删除（ *unmounted* ）。因此，将元素替换为完全不同于 DOM 树的东西代价可能会非常昂贵。幸运的是，这在现实世界中很少发生。

**记住 React 使用 ===（triple equals）来比较类型值是很重要的，所以它们必须是相同类或相同函数的相同实例。**

下一个场景要有趣的多，我们大部分人经常这么使用 React。

- **场景4： type 是一个组件。**

```jsx
// 更新之前
{ type: Table, props: { rows: rows } }
// 更新之后
{ type: Table, props: { rows: rows } }
```

------

但是没有任何改变啊！你可能会这么说，你错了。

------

> 值得注意的是，一个 component 的 render（只有类组件在声明时有这个函数）跟 ReactDom.render 不是同一个函数。”render” 一词在 React 的里确实有点过度使用。

如果 type 是一个函数或类的引用（即常规的 React 组件），并且我们启动了 tree diff 的过程，则 React 会尝试一直检查组件的内部逻辑，以确保 render 返回的值不会改变（防止副作用的措施）。对树中的每个组件进行遍历和扫描，但是在复杂的场景这个渲染过程成本会很高！

## 关注子节点

除了上述四种常见情况之外，当元素有多个子元素时，我们还需要考虑 React 的行为。假设我们有这样的元素：

```jsx
// ...
props: {
children: [
{ type: 'div' },
{ type: 'span' },
{ type: 'br' }
]
},
// ...
```

接下来我们想要交换这些元素的顺序

```jsx
// ...
props: {
children: [
{ type: 'span' },
{ type: 'div' },
{ type: 'br' }
]
},
// ...
```

接下来将会发生什么呢？

当进行 “diffing” 的时候，React 检查 props.children 里面的数组时，它开始将数组中的元素与之前看到的元素按照数组下标顺序进行比较：0 与 0，1 与 1，以此类推，每次比较，React 都会运用上述规则进行。 在我们的例子中，它会认为 div 变成了 span，应用之前的场景3，这并不是很高效的。想象一下我们从 1000 行的表格里，删除了第一行。 React 将会不得不更新其后的 999 行，因为按照索引来对比，他们的索引都发生了变化。

幸运的是，React 有一个内置的方法 [`built-inway `](https://reactjs.org/docs/lists-and-keys.html)[3]来解决这个问题。如果一个元素有一个 key 属性，元素可以通过 key 的值来比较，而不是使用索引。只要这个 key 是唯一的，React 便可以移动元素而不是从 DOM 树中删除它们，然后把它们再加回来（在 React 中叫挂载/卸载）。

```jsx
// ...
props: {
children: [ // 现在 React 将关注 Key，不再关注下标。
{ type: 'div', key: 'div' },
{ type: 'span', key: 'span' },
{ type: 'br', key: 'bt' }
]
},
// ...
```

> 译者注：在我们实际开发中，如果循环渲染同一个被复用的组件，使用相同 key 的数据渲染同一个组件，只会被渲染一次。

## 当 state 发生变化

到目前为止，我们只涉及到 React 哲学的 props 部分，却忽视了 state。这是一个简单的“有状态”组件：

```jsx
class App extends Component {
state = { counter: 0 }
increment = () => this.setState({
counter: this.state.counter + 1,
})
render = () => (<button onClick={this.increment}>
{'Counter: ' + this.state.counter}
</button>)
}
```

所以，在我们的 state 对象中，有一个 key 为 counter，点击按钮时它的值就会增加，并且按钮的文本也会改变。但是当我们这么做的时候，到底在 DOM 中发生了什么？哪部分将被重新计算和更新？这是我们需要思考的。

调用 this.setState 也会导致重新渲染，但不会影响整个页面，只会影响组件本身及其子节点。父节点和兄弟节点都不会受到影响。当我们有一个很庞大的树形结构时，只重绘它的一部分就很方便。

## 确定问题

我们准备了 [Demo ](https://iadramelk.github.io/optimizing-react-demo/dist/before.html)。 你可以看到最常见的问题。也可以在 [这里 ](https://github.com/iAdramelk/optimizing-react-demo)[4]查看源码。当然 [React 开发者工具 ](https://github.com/facebook/react-devtools)[5]也是需要，所以确保你的浏览器已经装好了它们。

我们首先要看的是，哪些元素？它们什么时候触发虚拟 DOM 的更新。在浏览器的开发工具中，打开 React 面板并选择 “Highlight Updates” 复选框：

![-w750](http://img12.360buyimg.com/uba/jfs/t20233/326/650298601/82260/e249b311/5b149c55Nf949d17c.png)（在 Chrome 中使用“突出显示更新”复选框选中 DevTools）

现在尝试向表中添加一行。如你所见，页面上的每个元素都有一个边框。这意味着，每当我们添加一行时，React 都在计算和比较整个虚拟 DOM 树。现在尝试点击一行中的 counter 按钮。你将看到在`state `变化之后虚拟 DOM 是如何更新的 —— 只有引用了 state key 的元素及其 children 受到了影响。

React 开发者工具会提示问题出在哪里，但不会告诉我们相关细节的信息：比如说所涉及的更新，是指 “diffing” 元素？还是挂载/卸载它们？要了解更多信息，我们需要使用 React 的内置分析器 [(profiler)](https://reactjs.org/docs/optimizing-performance.html#profiling-components-with-the-chrome-performance-tab)（注意它不能用于生产环境）

添加 `?react_perf `到应用的 URL，然后转到 Chrome DevTools 中的 “Performance” 标签。点击 “Record” 并在表格上点击。添加一些 row，更改一下 counter，然后点击 “Stop”。

![-w750](http://img14.360buyimg.com/uba/jfs/t21640/278/669366842/135750/9b2051ec/5b149c56Nfd84e6ef.png)（React DevTools的“Performance”选项卡）

在结果输出中，我们需要关注 “User timing”。放大时间轴直到看到 “React Tree Reconciliation” 组及其子项。这些是我们组件的所有名称，它们旁边都有 [update] 或 [mount]。

------

**我们的大部分性能问题都属于这两类问题之一。**

------

无论是组件（还是从它分支的所有东西）出于某种原因都会在每次更新时 re-mounted，其实我们不希望它 re-mounted （因为很慢），即使这些分支内容没有改变，我们却在大型应用分支的比对上花费很大开销。

## 优化：挂载/卸载

现在，当我们知道了 React 如何更新虚拟 DOM，并掌握了一些方法来看到其运行时背后到底发生了什么，我们终于准备好优化我们的代码了！首先，让我们来处理挂载/卸载。

如果你只是介意一个元素或组件在其内部有很多个子节点表示为数组，你可以获得非常显着的速度提升。

考虑如下示例：

```jsx
<div>
<Message />
<Table />
<Footer />
</div>
```

在我们的虚拟 DOM 中，上述代码将表现为：

```jsx
// ...
props: {
children: [
{ type: Message },
{ type: Table },
{ type: Footer }
]
}
// ...
```

这里有个简单的 Message 例子，一个 div 有一些文本，和一个超过 1000 行的庞大 Table。它们都包括在封闭的 div 中，所以它们被放置在父节点的 props.children 下，并且它们都没有 key。React 甚至不会通过控制台警告我们要给他们分配 key，因为 children 正在被 React.createElement 作为参数列表传递给父元素，而不是直接遍历一个数组。

现在我们的用户已读了一个通知，Message 组件从 DOM 树上移除后，剩下 Table 和 Footer 组件。

```jsx
// ...
props: {
children: [
{ type: Table },
{ type: Footer }
]
}
// ...
```

站在 React 的角度看，上述过程子节点是不断变化的：第0个子节点从 Message 组件现在变成了 Table 组件。这里没有 keys 来与之比较，所以它比较 types 时，又发现它们俩不是同一个 function 的引用，于是会把整个 Table 卸载，然后在挂载回去，渲染它的 1000 多行子节点数据！

因此，你可以添加唯一的 key（但在这种特殊情况下，使用 keys 并不是最佳选择），或者采用更智能的技巧：使用短路计算 [(short circuit boolean evaluation) ](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_Operators)，这是 JavaScript 和许多其他现代语言的特性。比如：

```jsx
<div>
{isShown && <Message />}
<Table />
<Footer />
</div>
```

虽然 Message 组件不会在画面显示，父元素 div 的 props.children 仍然有三个元素，children[0] 有一个值 false（一个布尔值）。请记住 true/false, null, undefined 是虚拟 DOM 对象 type 属性允许的值，我们最终得到了下面的结果：

```jsx
// ...
props: {
children: [
false, // isShown && <Message /> 结果为 false
{ type: Table },
{ type: Footer }
]
}
// ...
```

所以，Message 有或没有，我们的索引都不会变，当然，Table 仍然与 Table 进行比较（当 type 是一个引用类型时，对比一定会进行），但是仅仅对比虚拟 DOM 也比删掉 Dom 节点再从头创建它们来的快的多。

现在我们来看看更多高级的东西。我们知道你喜欢 HOC。HOC（高阶组件）是一个将组件作为参数，执行某些操作并返回不同功能的函数：

```jsx
function withName(SomeComponent) {
// 计算名称，可能代价很高...
return function(props) {
return <SomeComponent {...props} name={name} />;
}
}
```

这是一种非常常见的模式，但你需要小心。考虑：

```jsx
class App extends React.Component() {
render() {
// 在每个渲染上创建一个新的实例
const ComponentWithName = withName(SomeComponent);
return <SomeComponentWithName />;
}
}
```

我们在父组件的渲染方法中创建一个 HOC。当我们重新渲染组件树的时候，我们的虚拟 DOM 将如下所示：

```jsx
// 第一次渲染:
{
type: ComponentWithName,
props: {},
}
// 第二次渲染:
{
type: ComponentWithName, // 相同的名字，但是不同的实例
props: {},
}
```

现在 React 在 ComponentWithName 组件运行 diffing 算法，但此时同名引用了不同的实例，三等于（triple equals）失败，一个完整的 re-mount 会发生（整个节点换掉）注意它也会导致状态丢失， [如此处所述 ](https://github.com/facebook/react/blob/044015760883d03f060301a15beef17909abbf71/docs/docs/higher-order-components.md#dont-use-hocs-inside-the-render-method)。幸运的是，这很容易解决，你需要始终在 render 外面创建一个 HOC：

```jsx
// 仅仅创建一次一个新的实例
const ComponentWithName = withName(Component);
class App extends React.Component() {
render() {
return <ComponentWithName />;
}
}
```

## 优化：更新

所以，除非必要否则我们不建议 re-mount 。但是，对位于 DOM 树根部附近的组件所做的任何更改都会导致其所有子节点的 diffing 和 reconciliation。对于结构复杂的应用这资源开销也很大并且通常是可以避免的。

------

**有一种方法可以告诉 React 不要检查某个分支，因为我们确定它没有变化。**

------

这个方法叫 `shouldComponentUpdate `他是组件生命周期 [component’s lifecycle ](https://reactjs.org/docs/react-component.html#the-component-lifecycle)[6] 的一部分。这个方法会在组件的 render 和组件接收到 state 或 props 的值更新之前调用。然后我们可以自由地将它们与我们当前的值进行比较，并决定是否更新我们的组件（返回 true 或 false ）。如果我们返回 false，React 将不会重新渲染组件，也不会检查它的所有子组件。

通常，比较两个集合 props 和 state 一个简单的 *浅比较* （shallow comparison）就足够了：如果顶层的值不同，我们不必接着比较了。浅比较不是 JavaScript 的特性，但有很多这方面的工具 [utilities ](https://github.com/dashed/shallowequal)[7]。

现在可以像这样编写代码了：

```jsx
class TableRow extends React.Component {
// 将要返回 true 如果新的 props/state 与旧的不同
shouldComponentUpdate(nextProps, nextState) {
const { props, state } = this;
return !shallowequal(props, nextProps)
&& !shallowequal(state, nextState);
}
render() { /* ... */ }
}
```

你甚至不需要自己编写代码，因为 React 将这个特性内置在一个名为 React.PureComponent 的类中。它类似于 React.Component，只是在 shouldComponentUpdate 已经帮你实现了一个浅的 props/state 比较。

这听起来像是一件容易的事，只需在类定义的继承部分将 Component 改为 PureComponent，即可享受高效率。虽然不是很快！考虑这些例子：

```jsx
<Table
// map 返回一个新的数组实例，所以浅比较将失败
rows={rows.map(/* ... */)}
// 对象的字面量总是与前一个不一样
style={ { color: 'red' } }
// 箭头函数是一个新的未命名的东西在作用域内，所以总会有一个完整的 diffing
onUpdate={() => { /* ... */ }}
/>
```

上面的代码片段演示了三种最常见的 **反例** 。尽量避免它们！

------

**如果你能注意点，在 render 定义之外创建所有对象、数组和函数，并确保它们在调用期间不会变化 —— 你是安全的。**

------

你可以在 [updated demo ](https://iadramelk.github.io/optimizing-react-demo/dist/after.html)[8] 中观察 PureComponent 的效果，其中所有表格的行都是“纯净的”。如果你在 React DevTools 中打开 “Highlight Updates”，你会注意到只有表格本身和新行在行插入时渲染，所有其他行都保持不变。

但是，如果你迫不及待地想要使用 PureComponents 并在你的应用程序的任何地方使用它们，不要这么做。比较两组 props/state 开销也是不小的，对于大多数基本组件来说甚至都不值得：运行浅比较（shallow Compare）比 “diffing” 算法需要更多时间。

使用这个经验法则：纯组件适用于复杂的表单和表格，但它们通常会减慢按钮或图标等简单元素的速度。

感谢你的阅读！现在你已准备好将这些见解应用到你的应用程序中。你可以使用我们的小演示（无论是否使用 PureComponent）的存储库作为你的实验的起点。此外，请继续关注本系列的下一部分，我们计划涵盖 Redux 并优化你的数据以提高应用程序的总体性能。

## 译者之言：

整体文章翻译下来最大的收获就是：大部分特性在实际项目中都使用过，但是特性背后的细节原理确实较之前理解更加到位了，通篇下来作者由浅入深的指引我们把 React 的整个知识体系串讲了一遍。相信通读之后大家会有不一样的收获。

当然也建议大家阅读一下原文，阅读过程中如有任何不同见解，欢迎大家一起交流。

## 扩展阅读：

[1] 函数式 https://reactjs.org/docs/components-and-props.html#functional-and-class-components 
[2] diffing 算法 https://reactjs.org/docs/reconciliation.html 
[3] built-in way https://reactjs.org/docs/lists-and-keys.html 
[4] Demo 源码： https://github.com/iAdramelk/optimizing-react-demo 
[5] react-devtools：https://github.com/facebook/react-devtools 
[6] 生命周期： https://reactjs.org/docs/react-component.html#the-component-lifecycle 
[7] utilities： https://github.com/dashed/shallowequal 
[8] updated Demo： https://iadramelk.github.io/optimizing-react-demo/dist/after.html