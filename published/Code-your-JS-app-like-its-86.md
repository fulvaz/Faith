#[翻译]用上古思想写现代前端

原文信息
---
原文: [Code your JS app like it's 86](https://tech.polyconseil.fr/code-your-js-app-like-its-86.html)

原文作者：[Victor Perron](https://tech.polyconseil.fr/author/victor-perron.html)

原文创作时间：2016-09-28

翻译时间: 2017-01-10

翻译作者：[无法毕业的fulvaz](https://github.com/fulvaz)

Contact：fulvaz@foxmail.com

>译者注：
>这篇文章非常有趣，作者介绍了利用复古的的图形界面设计模式实现组件间解耦。但说起来古老，实际上说明了flux做了什么事情，以及为什么要有flux。

-------------------------------------

正文
----

利用过去的编程范式可以避免重写你的JavaScript应用。

## 太多时间花在了重写UI上面


我们会出于不同的原因去重写UI。

通常来说，我们会把原因归结到工具上 ---- 这很合理。

框架，库，包管理，guidelines，甚至是语言和元语言本身都变化地非常快，所以现在很难找到一个做应用的普遍适用标准。

更别提测试和测试的方法，那可以另外写一篇文章进行讨论。硬要进行测试，也只是靠眼睛去看他是否可以正常工作。

大多数情况，UI都会连同产品一起发布，用户会花钱买，然后在一个重写了数据库的关键更新后，用户会花钱再买一次。

整个行业的公司都会招新人毕业生，然后叫他们用最新的工具，在最短的时间组装出Web UI，最后，把UI作为产品的一部分卖掉。

最大的问题在于，没人会去考虑UI的长期支持（LTS）和维护 --- 这里的工作量差不多是要建一个生态系统。

我们可以做出改变。我们可以建立一个标准。我们可以专注于写Web UI时出现频率高的问题，然后解决这些问题。而且我们可以用古老的设计模式和编码策略去解决这些问题。

## 三个错误

下面看一个简单的React component例子。

```javascript
// FooComponent.js

import react from 'react'
import {ApiClient} from '../api_client'

var FooComponent = React.createClass({
  componentDidMount: function () {
    ApiClient.getTitle().then((data) => this.setState({title: data}))
  },
  handleClick: function (e) {
    e.preventDefault();
    bar_component.resetCoconuts()
  },
  render: function() {
    return (
      <div className="title">{this.state.title}/>
      <div className="button" onClick={this.handleClick}>
    )
  }
})
```

特别简单的一个例子，你可以看出上面代码有什么问题吗？

### 不恰当的导入和数据流
当应用的逻辑分散在你的组件和控制器中时，你也只能毫无办法地使用`隐式全局变量(hidden globals)`去组织你的代码。我用还是使用上面的例子，然后用其他方式重写这段代码。

首先，代码中先导入了`ApiClient`

```
import {ApiClient} from '../api_client'
```

这段代码将你的组件和数据源耦合了起来。这不是一种好的实践。这种设计至少有3个问题。

- 修改`ApiClient`会导致`FooComponent`跟着也要修改
- 测试`FooComponent`需要一个mock的`ApiClient`：比如一个HTTP后端
- 如果(在页面内)同时有两个`FooComponent`，那这个页面会想服务器提交两次请求。

但这些都不是主要问题。

主要问题是：这个API client还没有初始化。如果它需要一个base URL或者是一个token呢？

你的组件就需要去给这个API client提供参数。意思是你需要给这个组件提供一些选项的参数，然而这样就违反了关注点分离原则。

```
const myComponent = FooComponent.bootstrap('#anchor', {
    baseUrl: "https://xxxx",
    token: "MY_TOKEN",
    actualOption: xxxx,
})
```

这段代码中至少有两个非必要的参数（`baseUrl`和`token`）。这两个参数需要在测试的时候mock。

你有见过组件需要传URL才能工作吗？组件只需要数据。

### 这个组件依赖不可见的全局变量
其次，这段代码还依赖了全局的`bar_component`去处理点击事件和重置`cocounts`。 这种写法非常不好。

```
handleClick: function (e) {
  e.preventDefault();
  bar_component.resetCoconuts()
},
```

imports里面没任何东西可以提醒我们，这个组件依赖着在`window`对象的隐式全局变量`bar_component`.

另外，问题并不仅是因为它是全局的，这种写法还会引起其他的bug。比如说，`bar_component`在`handleClick()`函数的作用域中被定义了。（译注：那么bar_component重置的Cocount就是另外一个Cocount了）

下面列出了不同等级的麻烦：
* Level 1：你不能在没有`BarComponent`的情况下测试`FooComponent`.

* Level 10：`BarComponent`并不仅是一个依赖，它需要进行实例化


* Level 9001：这个实例需要保存在一个全局范围内。而不能通过显式声明，top-level, automatically-retrievable,或者是导入等方式来使用这个实例。

译注1：
此处top-level意思是通过查询依赖链的根部然后使用实例。

译注2：automatically-retrievable：自动查询依赖，[RPM的自动查询依赖](http://www.rpm.org/max-rpm/s1-rpm-depend-auto-depend.html)

对AngularJS用户：遇到这种情况的最常见原因是在`FooComponent`内注入`BarComponent`或者`ApiService`

当然，这些依赖都是可以mock的。（用angular特定的方式） 即便如此，他们依然是需要在定义在某处的全局变量。

他们间产生了耦合。


### 数据流不明确
一个非常常见的问题：代码各处都是数据查询（XHR，JSONP）

```
var FooComponent = React.createClass({
  componentDidMount: function () {
    ApiClient.getTitle().then((data) => this.setState({title: data.comment}))
  },
  // […]
})
```

这段代码中，除了我们刚才的耦合问题，还有数据流不清晰的问题，即你没法清晰地看出HTTP请求在哪，什么时候发出。

更糟糕的是，你部分组件可能依赖于未更新的请求（API可能会改变），而你的应用的其他部分依赖更新过的API请求。

如果通过XHR获得的（JSON内的）`comment`属性发生了改变，你就要在组件内部修改才能修正你的组件，这看起来并不太对。

这堆错误的实践累加起来，最终就会导致不久后的重写。你需要的是严格的关注点分离。实现的方法是提前计划好，尽可能准确地预估你的应用是做什么的，数据流是怎么样的。

然而，这里还有另一个范式。

### "main loop"
下面从介绍一个古老的设计模式开始。

回忆你写代码最开始要做什么：打开一个编辑器，创建一个`main()`循环，然后在某处输出『Hello World』

这看起来很简单，但在游戏和桌面软件领域，main循环是相关逻辑的骨架。

#### 在Windows API中
下面的简单代码来自Windows API例子：

```
LRESULT APIENTRY WndProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    PAINTSTRUCT ps;
    HDC hdc;

    switch (message)
    {
        case WM_PAINT:
            hdc = BeginPaint(hwnd, &ps);
            TextOut(hdc, 0, 0, "Hello, Windows!", 15);
            EndPaint(hwnd, &ps);
            return 0L;
        // Process other messages.
    }
}

int APIENTRY WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance,
                     LPSTR lpCmdLine, int nCmdShow)
{
    HWND hwnd;

    hwnd = CreateWindowEx(
        // parameters
    );

    ShowWindow(hwnd, SW_SHOW);
    UpdateWindow(hwnd);

    return msg.wParam;
}
```

非常复古的代码，对吧？

分析：WinMain函数使用几个参数创建了应用的window。其中WndProc回调函数负责处理各种事件：用户事件，重绘事件等等

一个main循环，一个事件循环。

就是这样。生存了20年，拥有数百万的应用
的Windows生态系统，只是基于简单的main函数和事件循环。

#### SDL API里
SDL API主要用来设计游戏。经常被当做轻量级的OpenGL。

下面是一个简单的app

```
#include <SDL2/SDL.h>
#include <iostream>


int main()
{
    SDL_Window* handle(0);
    SDL_Event events;
    bool end(false);

    if(SDL_Init(SDL_INIT_VIDEO) < 0)
    {
        SDL_Quit();
        return -1;
    }

    handle = SDL_CreateWindow("Test SDL 2.0", SDL_WINDOWPOS_CENTERED,
                              SDL_WINDOWPOS_CENTERED, 800, 600,
                              SDL_WINDOW_SHOWN);
    if(handle == 0)
    {
        SDL_Quit();
        return -1;
    }

    while(!end)
    {
        SDL_WaitEvent(&events);
        if(events.window.event == SDL_WINDOWEVENT_CLOSE)
            end = true;
    }

    SDL_DestroyWindow(handle);
    SDL_Quit();
    return 0;
}
```

不同的生态系统，但是东西还是那套东西。一个main入口把事情初始化了，一个事件循环负责处理用户输入和其他东西。

OpenGL应用极度简单，你可在这[这里](http://www.opengl-tutorial.org/download/)找到例子。

不敢相信对吧，就是main循环，初始化。这就是我们今天要说的：几乎所有类型的UI都是基于一个事件循环，然后应用的不同部分触发不同的事件。

那么，为什么大部分前端应用不使用同样的方法呢？

因为没人告诉过我们可以这么用。我们习惯用了jQuery，然后慢慢开始组建Angular组件，访问不知道定义在哪的全局变量。

### Javascript应用：Main loop model

我们已经知道了非常简单的Windows API例子、对游戏开发友好的OpenGL和SDL库。

在某种程度上说，一个web界面就是一个简单的图形应用。不同的是它用的是更现代的工具。

如果我们将我们的应用写成这样子

```
// main.js

import {ApiClient} from './api_client'
import {FooComponent} from './components/foo'
import {BarComponent} from './components/bar'

// Init the components
FooComponent.bootstrap($('#foo_component'), options)
const bar = new BarComponent(document.getElementByID('#bar_component')))

// Get the data
const api = ApiClient.authenticate(getTokenFromStorage())

api.fetchCoconuts(function (coconuts) {
  // Handle-based data passing
  bar.setCoconuts(coconuts)
  // Event-based data passing
  document.dispatchEvent(new Event('data_is_fetched', coconuts))
})

api.fetchTitle(function (data) {
  foo.setTitle(data.title)
})

// Event loop
document.addEventListener('foo:click', function () {
  alert('Foo component was clicked')
  bar.resetCoconuts()
})
```

```
// FooComponent.js

import react from 'react'

var FooComponent = React.createClass({
  handleClick: function (e) {
    e.preventDefault();
    // Notify other layers (simplistic, but works)
    document.dispatchEvent(new Event('foo:click'))
  },
  setTitle: function (str) {
    this.setState({title: str}))
  },
  render: function() {
    return (
      <div className="title">{this.state.title}/>
      <div className="button" onClick={this.handleClick}>
    )
  }
})
```

我们来认真看看上述代码，特别是`main.js`

我们获得了解耦和的组件，FooComponent和BarComponent都不需要知道数据请求的细节。

他们不需要知道，也不想知道。

他们所需要的是，请求自一个hardcoded fixture的`cocount`, 关键的是，`bar`对外暴露一个非常简单的`setCocount()`API函数。任何应用逻辑都可以在外部使用它。

事件在代码中广播（嫌low使用event bus也可以），不同的组件可以捕获事件然后进行处理。foo显式发送点击事件，在应用中监听这个事件，就可以实现通过点击`foo`然后重置`bar`的count。

这样，组件就真正地解除了耦合。他们可以随意重用，而不需要知道组件内部是如何实现的。

只有main循环里面需要写与app相关的业务逻辑。

注：这里没有使用全局变量，我们在main循环中利用了闭包。

### 总结
文中说的几个原则其实非常简单。在你的应用中使用这种简单的代码结构，然后继续使用这个代码结构。

尽管如此，我们还是看到了许多错误的设计模式。（或者根本没有设计模式）他们甚至连改善设计模式的想法都没有。

在Polyconseil公司，我们使用了就像上文中说的，经历时间洗礼的方法。我们使用了`make`，而不是`Grunt`，`Gulp`,`broccoli`,[详见](https://github.com/polyconseil/systematic)   

我们下次会讨论其他的编程范式，路由，还有一堆我们在Polyconseil和其他地方遇到的常见陷阱。

译注：你可能发发现这些解除耦合的方法很像：Flux


\- EOF -
