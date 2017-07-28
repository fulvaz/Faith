# [CSS-TRICKS]如何清除浮动?

注意: CSS-TRICKS的文章我推荐去看原文, 因为他们总是在更新. 本文已经注明了原文发表与本文翻译时间.

## 原文信息

原文：[The Clearfix: Force an Element To Self-Clear its Children](https://css-tricks.com/snippets/css/clear-fix/)

原文作者: [Chris Coyier](https://css-tricks.com/author/chriscoyier/)

原文最后更新时间时间：2017-02-26

翻译时间: 2017-06-24

## 翻译作者信息
翻译作者：毕业失业的fulvaz

联系方式：fulvaz@foxmail.com

FBI Warning: 翻译时优先关注原文的本意, 而非其实际内容, 介意请看原文

------------------------------------------

## 正文

以下代码在现代浏览器很好用

```
.group:after {
  content: "";
  display: table;
  clear: both;
}
```

将这段代码用在父元素上, 你就能把父元素内的子元素浮动全给清除了, 举个栗子:

```
<div class="group">
  <div class="is-floated"></div>
  <div class="is-floated"></div>
  <div class="is-floated"></div>
</div>
```

上古方法是把`<br style="clear: both;" />`这段放在父元素中的最末端, 这样不好记, 语义化低. 另一个上古方法是给父元素的CSS加`overflow: hidden;`, 问题是, 有些时候你就是想让父元素overflow不为hidden.

很明显这些都不给力啊! 那咋办呢

好了, 现在是各路大神对清除浮动的探索历史

首先是这个, 该版本目的是为了兼容尽可能久远的浏览器

```
.clearfix:after {
  visibility: hidden;
  display: block;
  font-size: 0;
  content: " ";
  clear: both;
  height: 0;
}
.clearfix { display: inline-block; }
/* start commented backslash hack \*/
* html .clearfix { height: 1%; }
.clearfix { display: block; }
/* close commented backslash hack */
```

嗯..很复杂, 然后是稍微好看一点的版本, 由Jeff Starr[提出](http://perishablepress.com/press/2008/02/05/lessons-learned-concerning-the-clearfix-css-hack/), 因为那时候已经没人用IE for Mac了 (译注: 应该08年, 了解[IE for Mac](https://zh.wikipedia.org/wiki/Internet_Explorer_for_Mac)), 所以代码中那段`backslash hack`可以去掉, 这时清除浮动代码如下:

```
.clearfix:after {
  visibility: hidden;
  display: block;
  font-size: 0;
  content: " ";
  clear: both;
  height: 0;
}
* html .clearfix             { zoom: 1; } /* IE6 */
*:first-child+html .clearfix { zoom: 1; } /* IE7 */
```

然后, 在Dan Cederholm的带领下, 用"group"作为类名成了潮流, 这样可以使html更优雅而且语义化. 另外, `content `属性不在需要空格, 用空字符串也可以(Nicolas Gallagher说的). 然后, 因为不需要任何文字, 那`font-size`也不需要了(Chris Coyier说的), 再次简化: (译注: zoom属性与hasLayout相关)

```
.group:after {
  visibility: hidden;
  display: block;
  content: "";
  clear: both;
  height: 0;
}
* html .group             { zoom: 1; } /* IE6 */
*:first-child+html .group { zoom: 1; } /* IE7 */
```

如果你不想支持IE 6或者IE 7, 那去掉后面两行也是可以的
> 2011-05-18更新: Nicolas Gallagher对上古浏览器的清除浮动又写了几篇文章: ["micro" clearfix](http://nicolasgallagher.com/micro-clearfix-hack/)与[additional stuff](http://nicolasgallagher.com/better-float-containment-in-ie/)
> 

```
.group:before,
.group:after {
  content: "";
  display: table;
} 
.group:after {
  clear: both;
}
.group {
  zoom: 1; /* For IE 6/7 (trigger hasLayout) */
}
```

当前最好用的方法? 看本文的第一段代码

将来的代码? 

```
.group {
  display: flow-root;
}
```

(译注: ....下次翻译文章应该先看看文章长度的)
