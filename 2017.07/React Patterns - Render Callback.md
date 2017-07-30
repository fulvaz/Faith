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

在React Native中创建一个list的方法是使用React Native核心库中的`ListView`。相比直接在子组件中渲染列表的项目，`ListView`定义了`renderRow`参数，这个参数是一个render callback，这个回调函数详细定义了如何根据给定的`dataSource`参数去渲染列表一行。这个例子中，`ListView `内部需要渲染一个列表时，我们用`render callback`模式来指定其内容的渲染样式。而且这样做能够优化性能，因为组件可以自行决定什么时候渲染列表项目。

```
<ListView
  dataSource={this.state.dataSource}
  renderRow={rowData => <Text>{rowData}</Text>}
/>;
```

优势
---
上文已经交代了这个模式的部分优点，最明显的优点其一是可以将渲染逻辑与数据获取逻辑解耦，另一个优点可实现按需要决定是否渲染组件。

还有一个好处是由于我们使用的是组件而且渲染函数依然是声明式的，阅读代码时很容易能看出来这个组件做了什么。

另外，与[高阶组件](https://facebook.github.io/react/docs/higher-order-components.html)不同，render callback不会污染 props，而在高阶组件中，注入的 props 会污染子组件的 props。而render callback只是注入一个参数。（译注：因此不会修改prototype）

缺点
---

没有设计模式是完美的，任何模式其实都是一种妥协。

`render callback`有一个问题，由于函数定义的位置是内联在  render 函数的元素中，如果待传函数作为参数的子组件在`shouldComponentUpdate`内做了优化，那么会产生冲突。另外因为这些函数每次都是一个新的实例，所以同一个 render callback 在不同的组件内是无法比较是否为同一个的。当然你可以将 render callback 放到组件实例里面去，不过这样就不能直接在 `render` 函数中直观地看出这个组件是怎么渲染的。然而，这都是很主观的观点。

当用于渲染的数据也被其他的生命周期的钩子函数依赖，那么使用这个模式是不合适的。这会导致副作用，因为在 render 函数没法将这些数据传到其他生命周期去，因此你还要在其他生命周期处理这个问题。

`render callback`是以参数的形式传递到另一个组件的 render 函数中的，因此在函数外没办法访问这些参数，父组件就更不可能访问了。如果你需要访问的话，那就以 props 的形式将参数传入另一个组件中，这样，你就是在解决如何通过 props 将注入一个普通组件的问题。另一个解决方法是使用高阶组件。

有一个常见问题需要注意，就是一不小心在 render callback 中使用了‘腐烂’ props或者 state。在对 props 或者 state 使用解构符号，然后将所得的值在 render callback 中以闭包的形式使用。来看个例子；


```
class Greeter extends React.Component {
  render() {
    const { name } = this.props;

    return (
      <Clock
        renderEverySecond={time => {
          return <h1>Hello {name} - The time is {time.toString()}</h1>;
        }}
      />
    );
  }
}

class Clock extends React.Component {
  constructor() {
    super();
    this.state = { element: null };
  }

  componentDidMount() {
    const { renderEverySecond } = this.props;

    setInterval(
      () => {
        this.setState({
          element: renderEverySecond(new Date())
        });
      },
      1000
    );
  }

  render() {
    return this.state.element;
  }
}
```

这看起来是一个别扭的例子，然而却常常出现在实践中。这个例子中，如果你想用不同的`name`参数多次渲染`Greeter`，那么`name`会一直停留在第一次传的参数中，每个`Greeter`的名字是一样的，因为`renderEverySecond`会在`Clock`组件中被调用，就算后来`Greeter`将新的`renderEverySecond `与`name`传入了组件中，结果依然不变。

解决方式之一是不在`componentDidMount `中使用对象解构符，这样就能使`Clock `使用最新的`renderEverySecond `。


```
class Clock extends React.Component {
  // ...

  componentDidMount() {
    setInterval(
      () => {
        this.setState({
          element: this.props.renderEverySecond(new Date())
        });
      },
      1000
    );
  }
}
```

另一个解决方法是使`renderEverySecond `使用最新的`name`，同样是避免了解构符号，而是使用最新的`props`:

```
class Greeter extends React.Component {
  render() {
    return (
      <Clock
        renderEverySecond={time => {
          return (
            <h1>Hello {this.props.name} - The time is {time.toString()}</h1>
          );
        }}
      />
    );
  }
}
```

这个模式最后一个缺点是，就算传入 render callback 的是静态参数，这段代码也不会获得任何优化，因为 render callback 们是在渲染时动态组合的。

结论
---
本文介绍了`render callback`的概念，如何使用，在开发基于React程序的使用场景。本文还介绍了其在开源库中的使用，分析了他的优缺点，当然主要是和与它相似的模式-----高阶组件，进行了对比。

这看起来对大多数情况来说，选择`render callback`还是高阶组件都是个人倾向的问题。这两种模式的目的是一样的，只是各自的妥协有所不同。当然了，你可以在app中同时使用两种模式，然他们解决不同的问题。二者其实并不冲突。

希望这篇文章有所帮助，能让你的思路清晰一些，至少没有给你传达错误的信息。欢迎去原文评论，指出文章错漏，或者是如果你有关于模式优劣的想法可在评论留言与作者探讨。


-------------------------------------------------------

### 原文信息

原文：[React Patterns - Render Callback](https://leoasis.github.io/posts/2017/03/27/react-patterns-render-callback)
原文作者: leoasis
原文最后更新时间时间：2017-03-27

翻译时间: 2017-07-27
翻译作者：fulvaz \ fulvaz@foxmail.com

请将翻译的错误直接扔我邮箱，用力地鞭笞我吧 ：）

WARNING: 推荐看原文，因为原文可能会随着时间更新，比如CSS-TRICKS的文章。我已经标注了翻译时间和作者创作/更新时间