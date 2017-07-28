[译]为什么要写好Git Commit信息
===

### 原文信息

原文：[The Why: A Better Git Commit Message](https://8thlight.com/blog/dariusz-pasciak/2016/10/31/the-why-a-better-git-commit-message.html?utm_source=wanqu.co&utm_campaign=Wanqu+Daily&utm_medium=website)

原文作者: [Dariusz Pasciak](https://8thlight.com/blog/dariusz-pasciak/)

原文创作时间：31 Oct 2016

### 翻译作者信息
翻译作者：在想办法毕业的fulvaz

联系方式：fulvaz@foxmail.com


-------------------

我最近在修复一个第三方整合的issue。这个issue不知道为什么忽然间就出现了，问题的源头是页面中的一个iframe行为异常。

我直觉先去`git blame`含有iframe的代码，先看看上一次变动是什么时候，改了什么。非常简单的改动：
```
- <iframe src="https://fake-3rd-party.com/widget?code=12345" />
+ <iframe src="https://fake-3rd-party.com/embed/WXYZ" />
```

很明显可以看出来，整合方式的改变导致了这个问题。我去看了commit信息，想找到更多信息。然后commit信息是这样的：

```
Author: Some Guy <some.guy@some.domain>

    changed iframe src
    
```

这是在逗我吗！什么都看不出来啊！

好吧，我想说的是。

写commit信息前，先花一点时间思考一下这个信息是否内容丰满。我们能不能添加多一点可以让人理解这次commit做了什么的背景内容，加一个链接，或者其他也可以。

如果真的被难住了，不知道该如何写出有用的commit信息，那么可以问自己："为什么我要改？" 回答这个问题至少能表明这个改动的动机，从某种程度上讲，这条commit信息算是内容丰满的。最起码，这条信息可以作为将来查问题原因的起点。

从今天起把"原因"放到commit信息里面，然后坚持吧

\begin{ E=hi }\end
