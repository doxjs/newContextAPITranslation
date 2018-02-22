## 简介
 介绍用于解决现有局限性的全新Context API。

##  基本实例
```js
type Theme = 'light' | 'dark';
// Pass a default theme to ensure type correctness
//传递默认的主题确保类型的正确
const ThemeContext: Context<Theme> = React.createContext('light');

class ThemeToggler extends React.Component {
  state = {theme: 'light'};
  render() {
    return (
      // Pass the current context value to the Provider's `value` prop.
      // Changes are detected using strict comparison (Object.is)
      //传递现有的Context的值给Provider的props上的`value`属性
      //数据变化使用Object.is来进行严格比较
      <ThemeContext.Provider value={this.state.theme}>
        <button
          onClick={() =>
            this.setState(state => ({
              theme: state.theme === 'light' ? 'dark' : 'light',
            }))
          }>
          Toggle theme
        </button>
        {this.props.children}
      </ThemeContext.Provider>
    );
  }
}

class Title extends React.Component {
  render() {
    return (
      // The Consumer uses a render prop API. Avoids conflicts in the
      // props namespace.
      //消费层使用一个渲染的prop API，以避免与props命名空间冲突
      <ThemeContext.Consumer>
        {theme => (
          <h1 style={{color: theme === 'light' ? '#000' : '#fff'}}>
            {this.props.children}
          </h1>
        )}
      </ThemeContext.Consumer>
    );
  }
}
```
## 动机
通常情况下，React里面的数据是按照top-down（parent to child）的顺序，通过props来传递的。但有些时候，跳过多个抽象层级来传递一些值往往是很有用处的。就如例子中所传递的UI主题参数一样。很多组件都依赖于此，可你却不想在每一层组件上都使用prop来向下传递。

React里面的Context API正是为了解决这一问题所诞生的。在本文中，我们将祖先组件成为Provider（提供者），把孩子组件称为Consumer（消费者）。

## 现有版本的Context API的缺点

### shouldComponentUpdate会组织Context的更改
现在的Context API的主要问题就是如何与`shouldComponentUpdate`交互。如果一个中间组件使用了`shouldComponentUpdate`来操作，那么它的下方组件在没有等待更新的时候，React将认为整个子树都没有发生改变。如果子树包含了Context的消费者，那么消费者将不会收到任何最新的Context。换句话来说，Context的更改将不会在`shouldComponentUpdate`返回为false上的组件上传播。

在React应用中，`shouldComponentUpdate`是经常使用的优化操作方式。在共享组件与开源库中它的使用往往很频繁。在实践中，这意味着Context在广播变化的时候是不可靠的。

## 将用户空间复杂化

现在，开发者们通过使用订阅来绕过了`shouldComponentUpdate`的问题
* 提供者从当事件发生器，它追踪最新的Context，并且当它发生更改时通知订阅了的用户。
* 消费者使用Context API来访问事件发生器（这种方法很好，因为事件发生器本身并不会发生更改）。
* 消费者向提供者注册一个事件监听器。
* 当提供者出发了一个变化事件，消费者会收到通知并调用`setState`来更新并触发重渲染。订阅被开源软件广发使用，例如`Redux`和`React Broadcast`。它很有用，但它也有一些明显的缺点：
* 不符合人体工程学。由于Context使用的普遍性，正确的实施起来应该不会特别困难。
* 启动成本。为每个消费者订阅的花费很高，特别是因为它们在初始挂载的时候并未使用。
* 鼓励突变（注：这里原文是Encourages mutation，不知道怎么翻译，直接直译）和一些不常用的但会在异步模式下造成一定的BUG的模式。
* 相同的代码在每个库中都会被复制一次，这样增加了包的大小。
最根本的问题在于，核心功能的所有权和责任已经从框架转移到了用户身上。

## 这个提案的主要目的
* 零成本（或接近于零）的初始安装、提交和卸载，根据需要取舍的更新成本。
* 简单使用的API
* 静态的类型
* 鼓励异步友好的实践，例如不可变性。
* 组织费非理想的做法，例如事件发生器与变异。
* 消除用户级代码中重复的复杂性。

## 详细的设计

介绍全新的组件类型：`Provider`和`Consumer`：
```js
type Provider<T> = React.Component<{
  value: T,
  children?: React.Node,
}>;

type Consumer<T> = React.Component<{
  children: (value: T) => React.Node,
}>;
```
`Provider`和`Comsumer`是成对出现的，对于每一个`Provider`，都会有一个对应的`Consumer`。一个`Provider`-`Consumer`组合是用`React.createContext()`来产生的：
```js
type Context<T> = {
  Provider: Provider<T>,
  Consumer: Consumer<T>,
};

interface React {
  createContext<T>(defaultValue: T): Context<T>;
}
```
`createContext`需要一个默认值来确保类型的正确。

请注意，即使任何提供者与任何消费者的值的类型相同，它们也不能联合使用。它们必须是同一个`createContext`产生的结果。

`Provider`在`props`上接收一个`value`，无论嵌套的深度如何，它都能被提供者的任何匹配的消费者所访问。
```js
render() {
  return (
    <Provider value={this.state.contextValue}>
      {this.props.children}
    </Provider>
  );
}
```
为了更新Context的值，父级重新渲染并传递一个不同的值。Context的变化将被检测到，检测所使用的是`Object.is`来比较的。这意味着鼓励使用不可变或持久的数据结构。在典型的场景中，通过调用提供者父级的`setState`来更新Context。

消费者使用一个`render prop API`：
```js
render() {
  return (
    <Consumer>
      {contextValue => <Child arbitraryProp={contextValue} />}
    </Consumer>
  )
}
```
请注意上面这个例子，Context的值可以传递给子组件上的任意prop。`render prop API`的优点就是避免了破坏prop的命名空间。

如果一个`Consumer`没有提供一个匹配的`Provider`作为它的祖先，它会接收搭配传递给`createContext`的默认值，确保类型安全。

## 缺点

### 依赖于Context的值的严格比较

该提案使用严格（参考）比较来检测对Context的更改。这部分鼓励使用不可变性或持久的数据结构。但许多常见的数据源都依赖与突变。例如`Flux`的某些实现，甚至像`Relay Modern`这样的更新的库。


但是，与异步渲染结合时，突变存在固有的问题，主要与撕裂有关。 对于依赖变异的体系结构，开发人员要么决定某种程度的撕裂是可以接受的，要么演变为更好地支持异步。 无论如何，这些问题并不仅限于Context API。（From Google Translation）

一些依赖于突变的库的一个技巧就是克隆产生一个全新的外部容器（或者甚至只是在他们之间交替）。React将检测到一个新对象的引用并且触发一个更改。

## 每个Consumer只有一个Provider

建议的API只允许消费者从单一提供者类型读取值，这与当前的API不同，后者允许消费者从任意数量的提供者类型读取。

解决方法是使用合成消费者（待定）：
```js
<FooConsumer>
  {foo => (
    <BarConsumer>
      {bar => (
        // Render using both foo and bar
        <Child foo={foo} bar={bar} />
      )}
    </BarConsumer>
  )}
</FooConsumer>;
```
大多数围绕上下文的抽象已经使用了类似的模式。

## 备用方案

### setContext

我们可以使用像`setState`一样工作的`setContext` API，而不是依赖于引用相等来检测对上下文的更改。
但是，如果不考虑实现的开销，这个API只有在与突变结合使用时才有价值，我们专门致力于阻止这种突变。

### 将context传递给shouldComponentUpdate

一个说法是，我们可以通过将context作为参数传递给该方法来避免shouldComponentUpdate问题，将传入的Context与前一个Context进行比较，如果它们不同，则返回true。 问题是，与prop或state不同，我们没有类型信息。 上下文对象的类型取决于组件在React树中的位置。 您可以对这两个对象执行浅层比较，但只有在我们假定这些值是不可变的时才有效。 如果我们假设这些值是不可变的，那么React可能会自动进行比较

### 基于类的API
我们可以使用我们现在使用的基于类的API，而不是render prop：
```js
class ThemeToggler extends React.Component {
  state = {theme: 'light'};
  getChildContext() {
    return this.state.theme;
  }
  render() {
    return (
      <>
        <button
          onClick={() =>
            this.setState(state => ({
              theme: state.theme === 'light' ? 'dark' : 'light',
            }))
          }>
          Toggle theme
        </button>
        {this.props.children}
      </>
    );
  }
}

class Title extends React.Component {
  static contextType = ThemeContext;
  componentDidUpdate(prevProps, prevState, prevContext) {
    if (this.context !== prevContext) {
      alert('Theme changed!');
    }
  }
  render() {
    return (
      <h1 style={{color: this.context.theme === 'light' ? '#000' : '#fff'}}>
        {this.props.children}
      </h1>
    );
  }
}
```
这个API的优点是您可以更轻松地访问生命周期方法中的上下文，可能避免在树中需要额外的组件。

但是，尽管增加React树的深度会带来一些开销，但使用特殊组件类型的优势在于，为消费者扫描树会更快，因为我们可以快速跳过其他类型。 使用基于类的API时，我们必须检查每个类组件，它稍微慢一些。 这足以抵消额外组件的成本。

与类API不同，render prop API还具有与现有Context API充分不同的优势，我们可以在过渡时期支持这两个版本，而不会造成太多混淆。

（注意，原文部分接下来的一些章节并未翻译，这些章节涉及的内容是React官方的一些计划）

## 未解决的问题

### 当Consumer没有匹配Provider时，是否应该发出警告

对于依赖默认Context的值，这是很有效的。在很多情况下，这种错误应该是开发者所造成的，因此我们可以打印警告出来。为了关闭这个，你可以传递true给`allowDetached`。
```js
render() {
  return (
    // If there's no provider, this renders with the default theme.
    // `allowDetached` suppresses the development warning
    <ThemeContext.Consumer allowDetached={true}>
      {theme => (
        <h1 style={{color: theme === 'light' ? '#000' : '#fff'}}>
          {this.props.children}
        </h1>
      )}
    </ThemeContext.Consumer>
  );
}
```
### 将displayName参数添加到createContext以更好地进行调试

对于警告和React DevTools，如果提供者和消费者具有displayName，则会有所帮助。 问题是这是否应该要求。 我们可以使其成为可选项，并使用Babel变换自动添加名称。 这是我们用于createClass的策略。

### 其他
* 消费者是否应该使用`children`作为prop或命名prop？
* 我们应该多快弃用和删除现有的上下文API?
* 需要运行基准测试来确定这将是多快。
* 启发式缓存。
* 此功能的高优先级版本用于动画。 （可以单独提交。）

