注: 原文的代码部分直接引用自github，代码部分可能会更新

-----------------------------

这篇文章主要关于使用React开发程序时的常见设计模式。写这篇文章的主要原因是我想更深入地研究这些设计模式，对这些设计模式有更深刻地理解。比如这些模式的适应场景，使用这些模式需要进行哪些妥协. 希望这篇文章会对你有所帮助。

那我们首先说明"Render callbacks"这一模式。我感觉这个名字Twitter上的 [Ryan Florence](https://twitter.com/ryanflorence) 取的, 他说另一个名字 “function as child” 不好，“function as child” 不完全准确，因为这个模式并不仅仅可用于于`chidren`属性.

## 什么时候用

当你需要从父组件中提取一些功能出来到子组件中的时候。然而那些功能还依赖其他组件的状态，而且仅仅传递 props 是不够的。子组件要做执行一个函数，但是函数内部的某些逻辑需要父组件提供额外的信息才能完成。

Basically what you want is a way for the parent to provide some logic to the child, and the child have the control on how to execute that logic.

简单来说，你需要的是一种父组件可以向子组件传递函数的方式，而子组件可以决定怎么执行那个逻辑。

## 是什么

给子组件提供一个函数的 prop，而这个函数会在子组件内部的某个时刻被调用。这样的 prop 就是 “render callback”。这个函数在子组件执行，那么传递什么参数完全可以由子组件决定，函数的执行结果由子组件状态不同而不同。是否调用这个函数完全取决于子组件，取决于子组件的内部逻辑。

```
class Parent extends React.Component { 
	 render() {
	 	return <Child foo={bar => <div>Hello {bar}!</div>} />; 
	 } 
 } 
```

在子组件中，将参数按需要作为函数调用，所需要的参数传递给函数。 （译注：墨迹啊。）

```
class Child extends React.Component {
  // Do some complex stuff in the lifecycle...

  render() {
    if (someLogic) {
      return <div>Nothing to do here!</div>;
    } else {
      return this.props.foo(this.state.barCalculatedSomehow);
    }
  }
}
```

下面用例子说明。

## 例子：组件内获取远程数据

Let’s say you have a component that needs to fetch some data in order to render it somehow. Maybe you’d do something like this:

设想一个场景，某个组件内需要渲染一些数据，而这些数据需要先获取。那么常见做法可能如下：


```
class StuffList extends React.Component {
  constructor() {
    super();
    this.state = {
      loading: true,
      error: null,
      data: null
    };
  }

  componentDidMount() {
    fetch(`/user/${this.props.userId}/stuff`).then(
      data => this.setState({ loading: false, data }),
      error => this.setState({ loading: false, error })
    );
  }

  render() {
    if (this.state.loading) return <Loading />;
    if (this.state.error) {
      return <ErrorMessage error={this.state.error} />;
    }

    return (
      <ul>
        {this.data.map(thing => (
          <li key={thing.id}>
            <h2>{thing.name}</h2>
            <p>{thing.text}</p>
          </li>
        ))}
      </ul>
    );
  }
}
```

我们会利用组件的生命周期与 state 去完成所需的功能。首先不管数据是否读取完毕，还是请求有错误，或者是数据本身有错，当请求完成后，我们都会把数据存储到 state 里面。然后，在`componentDidMount`的生命周期的钩子函数中根据所需要的 props 请求数据，在请求成功或者失败时更新 state。

> 注意：如果需要一个更完整的例子，还需要在`componentWillReceiveProps` 与 `componentDidUpdate` 的钩子函数中加点代码，因为当与获取数据相关的 props （本文例子是`userId`）有变化时，需要重新发起一次请求（或者是取消/无视前一次还在进行的请求）。本文为了方便讨论就不加这部分代码了。

最后，我们让 render 函数用这些 state 去渲染加载界面，或者是个错误信息，再不然就是实际的数据请求结果。

来总结一下这个组件需要做什么。这个组件有多个职责：处理与数据获取相关的逻辑、复制加载与错误信息、把 state 渲染出来。我们希望将获取数据、处理不同 state、渲染加载与错误状态的逻辑抽取出来。我们只是关心如何在组件中渲染成功加载的数据。(译注：事实上文中例子只是把渲染数据的逻辑抽取出来)

我们原先会怎么使用这个组件呢？我们当然是将所需要的参数以 props 的传递给子组件，然后获取数据，这个例子中，参数是`userId`:

```
// I want my render logic to happen somewhere,
// but I need access to `this.state.data` that now lives
// inside `FetchStuff`
<FetchStuff userId={this.props.userId} ... />
```

Now, in order to _provide_ the child component with the necessary info to render the data when the request is successful, we’ll provide a render callback prop. That render callback should receive the data as its first parameter:

那么现在为了给子组件提供所需要的数据以在请求成功时渲染页面，我们会为子组件提供一个 render callback 作为 prop。这个render callback会将请求所得数据作为自己的第一个参数，子组件的用法如下：

```
<FetchStuff
  userId={this.props.userId}
  renderData={data => (
    <ul>
      {data.map(thing => (
        <li key={thing.id}>
          <h2>{thing.name}</h2>
          <p>{thing.text}</p>
        </li>
      ))}
    </ul>
  )}
/>;
```

由以上的用法，`FetchStuff`的实现如下:

```
class FetchStuff extends React.Component {
  constructor() {
    super();
    this.state = {
      loading: true,
      error: null,
      data: null
    };
  }

  componentDidMount() {
    fetch(`/user/${this.props.userId}/stuff`).then(
      data => this.setState({ loading: false, data }),
      error => this.setState({ loading: false, error })
    );
  }

  render() {
    if (this.state.loading) return <Loading />;
    if (this.state.error) {
      return <ErrorScreen error={this.state.error} />;
    }

    return this.props.renderData(this.state.data);
  }
}
```

注意看`this.props.renderData(...)`是怎么调用与传递参数的方法。这就是这个模式的实践。我们将全部不想在`StuffList`出现的功能提取了出来，使这个组件变成一个接受函数作为参数的可复用组件，这个组件可以按照所接受函数所描述的形式渲染获得的数据。

## 例2: Tooltip

再设想一个场景，你需要一个tooltip, 这是实现的一个方法:

```
<TooltipTrigger tooltipContent={<span>This is the tooltip</span>}>
  <div>Hover me!</div>
</TooltipTrigger>;
```

我们在`TooltipTrigger`组件中包括了在tooltip中内容的子元素。tooltip的内容将会作为一个 prop 传入组件中，这个 prop 接受一个react组件。`TooltipTrigger`的简单实现如下：

```javascript
class TooltipTrigger extends React.Component {
  constructor() {
    super();

    this.state = { hovering: false };
  }

  handleMouseEnter = () => {
    this.setState({ hovering: true });
  };

  handleMouseLeave = () => {
    this.setState({ hovering: false });
  };

  render() {
    return (
      <div
        onMouseEnter={this.handleMouseEnter}
        onMouseLeave={this.handleMouseLeave}
        style={{ position: "relative" }}
      >
        {this.props.children}
        {this.state.hovering &&
          <div
            style={{
              position: "absolute" /* plus some other styles to position the tooltip */
            }}
          >
            <Tooltip>{this.props.tooltipContent}</Tooltip>
          </div>}
      </div>
    );
  }
}
```

注意我们这里只在用户鼠标在`div`上停留时在会渲染`Tooltip`组件，其他情况不进行渲染。很简单对吧？

我们来思考一下这种实现方法会有什么问题。如果tooltip的内容极度复杂，我们希望在显示tooltip的时候才显示这个组件该怎么办？又或者是你还没有渲染tooltip内容的数据，需要等会再创建内容该怎么办呢？最理想的情况是可以用延迟加载的方式去提供tooltip所需内容，那么元素只会在`TooltipTrigger`需要的时候才创建。看起会是用_render callback_的场景。

如果我们不将内容放在render callback中，那么tooltip的内容会马上创建，就算tooltip组件还不打算渲染他们。虽然这些只是非常轻量的对象，他们只是描述了UI该长什么样，然而他们还是浪费了资源。另外，如果内容里面所包含的数据我还没获取到，不放在回调函数里面还会导致运行时错误，特别是我们需要访问的属性嵌套在另一个属性当中。（译注：比如访问`obj.a`，直接访问`obj`输出结果可能直接`undefined`，而`obj.a`则是运行时错误）

那来改下代码，不传react组件，改为传render callback作为tooltip的内容。代码如下：

```
<TooltipTrigger
  tooltipContent={() => (
    <span>Something very complex with {data.we.may.not.yet.have}</span>
  )}
>
  <div>Hover me!</div>
</TooltipTrigger>;
```

那么`TooltipTrigger `的实现修改如下：

```
class TooltipTrigger extends React.Component {
  // ...

  render() {
    return (
      <div
        onMouseEnter={this.handleMouseEnter}
        onMouseLeave={this.handleMouseLeave}
        style={{ position: "relative" }}
      >
        {this.props.children}
        {this.state.hovering &&
          <div
            style={{
              position: "absolute" /* plus some other styles to position the tooltip */
            }}
          >
            <Tooltip>
              {this.props.tooltipContent() /* Notice the change here! */}
            </Tooltip>
          </div>}
      </div>
    );
  }
}
```

这里我们使用render callback去渲染鼠标在指定内容上悬停时所显示的内容。`TooltipTrigger`的实现中唯一需要变化的就是使用`this.props.tooltipContent`，这是个获取渲染内容的函数。

这个例子中，与先前的例子相比，我们用render callback模式将是否渲染一个元素的决定权从tooltip组建中移交到另一个组件里面。

## 社区中的例子

还有很多开源库中用到了这个模式，下面去部分进行介绍，并对其应用进行简短的解释。

### React router

https://github.com/ReactTraining/react-router

事实上这个库用render callback把以上两个例子需要做事情都做了。 `Route`组件获取本地数据与历史记录中的“查询字符串参数”（query string params），然后将这些数据传到render callback形式的函数中。另一方面，是否调用这些函数取决于路由路径是否匹配

```
<Route path="/some/url" render={() => <h3>Hello.</h3>} />;
```

### React Measure

https://github.com/souporserious/react-measure

这个开源库用于测量一个元素的“尺寸” ---- width、height、top、bottom等等，然后将这些属性以参数的形式传到render callback中，这样父组件可以对元素做各种与“尺寸”相关的处理。

```
const ItemToMeasure = () => (
  <Measure>
    {dimensions => (
      <div>
        Some content here
        <pre>
          {JSON.stringify(dimensions, null, 2)}
        </pre>
      </div>
    )}
  </Measure>
);
```

这段代码有意思的地方是，render callback的参数是以`children`的形式传递进来的。这看起来有些特殊，但在React中，`children` prop 只是另一个 prop，只是这其中有一个语法糖，可以将函数放到JSX tag中，以实现传参。

### React Media

https://github.com/reacttraining/react-media

与先前的想法一样，只是这里所传进render callback的参数取决于 prop 与所设定的 media query是否匹配。

```
class App extends React.Component {
  render() {
    return (
      <div>
        <Media query="(max-width: 599px)">
          {matches =>
            matches
              ? <p>The document is less than 600px wide.</p>
              : <p>The document is at least 600px wide.</p>}
        </Media>
      </div>
    );
  }
}
```

### React-Motion

https://github.com/chenglou/react-motion

一个让你可以创建动画的库。用法是将动画的配置以参数的形式传到`Motion`组件中，然后将 render callback 以 children 的形式传递到组件中。这个回调函数会以动画运行所需要的中间值为参数不停地被调用，然后将全部中间状态渲染到UI中。

```
<Motion defaultStyle={{ x: 0 }} style={{ x: spring(10) }}>
  {value => <div>{value.x}</div>}
</Motion>;
```

### React Native’s ListView

https://facebook.github.io/react-native/docs/listview.html

在React Native中创建一个list的方法是使用React Native核心库中的`ListView`。相比直接在子组件中渲染列表的项目，`ListView`定义了`renderRow`参数，这个参数是一个render callback指定了


-------------------------------------------------------

### 原文信息

原文：[React Patterns - Render Callback](https://leoasis.github.io/posts/2017/03/27/react-patterns-render-callback)
原文作者: leoasis
原文最后更新时间时间：2017-03-27

翻译时间: 2017-07-27
翻译作者：fulvaz \ fulvaz@foxmail.com

请将翻译的错误直接扔我邮箱，用力地鞭笞我吧 ：）

WARNING: 推荐看原文，因为原文可能会随着时间更新，比如CSS-TRICKS的文章。我已经标注了翻译时间和作者创作/更新时间