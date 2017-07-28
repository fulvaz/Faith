[译]为什么不用Firebase
===

### 原文信息

原文：[Reasons Not To Use Firebase](https://crisp.im/blog/why-you-should-never-use-firebase-realtime-database/?utm_source=wanqu.co&utm_campaign=Wanqu+Daily&utm_medium=website)

原文作者: [Baptiste Jamin](https://crisp.im/blog/author/baptiste/)

原文创作时间：18 September 2016

### 翻译作者信息
翻译作者：在面试中的fulvaz

联系方式：fulvaz@foxmail.com


-------------------
![fire](https://crisp.im/blog/content/images/2016/09/3879581414_16f8e66499_b.jpg)

译注：Firebase即国外版的Leancloud。 

译注2：文中`MVP`并非最有价值球员，而是[Minimum viable product](https://en.wikipedia.org/wiki/Minimum_viable_product)：MVP指只有基本功能的产品，一般开发周期短，只有核心功能。创业公司可以用这种方法低风险验证自己的想法是否可行。

按今天的标准，我们要做可实时交互应用。在[Crisp](https://crisp.im/)， 我们从上线开始一直使用Firebase。现在已经九个月了，我们见识了从美好未来到永恒梦魇的全过程。

要说明的是，本文的范围只是在生产环境中使用Firebase的实时数据库。我们也认同一点，在Hackthon、需要做Demo或者是你自己的需要时，Firebase会是一个非常好的选择。

Edit：Firebase其实满足了我们的需求，而且非常适用于我们所面临的问题。Firebase也是一个优秀的团队。但是呢，在我们的例子中，我们的需求非常特别，所以Firebase（pre-2016 Google I/O version）才无法满足我们。尽管如此，Firebase还是一个优秀的方案。前提是你需要的只是存储用户数据和解决实时（real-time）问题。


我们的项目
---
### 为什么我们一开始使用Firebase？
Crisp是个非常简单的聊天应用，关注点在用户体验上。我们需要一个低成本的方案，所以呢，打算使用Socket.IO作为后端实现网页在线聊天，同时，使用Firebase实现控制后台，全部都基于AMQP协议同步到后端。

我们用这种混合架构只用了三周就做出了MVP  --- 包括在线聊天和控制后台。我们为同时不同用户提供了这两项服务，还将这两个Feature和微服务联系了起来。这个无服务端架构（server-less）、MVP式的控制台让我们更加专注UI和UX。从这点上看，用Firebase是非常合适的。

我们快速开发出了产品并将其推出了市场 ---- MVP应有样子。

### 然后就悲剧了
最开始一个月运行得很棒，然后就是悲剧。最开始我们上了ProductHunt的推荐。产品的用户量变多，随着流量的增加，我们遇到了扩容问题（scale issues）。

很多时候，Firebase都是一个噩梦。它拖慢了我们的步伐。我们很难增加新功能，在发布新功能前，我们必须认真多次检查。

我们也发现了Firebase的技术限制。当我们的月流量超过100GB时，我们遇到了性能上的问题。

不用Firebase的10个理由
---
### 1. 意大利面条代码
无服务端架构（server-less），并不意味着可以少写代码（code-less）。用Firebase意味着你全部服务端逻辑都运行在浏览器或者是移动端中。

在大多数app中，都会有这样的功能：发欢迎邮件、处理图像（比如头像等）、处理支付和核心业务逻辑。你不会想要把这些功能放在客户端去处理。因为这么做会使威胁你的业务安全。

如果你用Firebase，你需要用些小手段（hacked）解决这些问题。就是说，你需要为你的web-app写更多的代码。如果你还有一个手机app，那维护起来就是个噩梦。

想想发布的例子：客户端app升级，数据库逻辑也会跟着变（译者：真的？）。那那些旧版本的app怎么办？我们强制他们去升级吗？

### 2. 将Firebase和微服务整合是场灾难

我们在Crisp上使用了微服务。在多数用例中，我们需要查询数据库，获取用户信息，ID等等。

Firebase既能在客户端（如Web、移动app）用，也能在后端（比如NodeJS）中使用。你可能用你的后端从Firebase中请求数据，你必须停止这么做，因为真的很慢。

我们用Redis去缓存Firebase的查询结果，那就是说，我么要保持数据同步。但是我们做缓存的微服务（或者是是连接Firebase的微服务）出现了内存溢出问题。Firebase提供的NodeJS库是在内存中缓存数据，而且它似乎没有释放内存，没有被引用的变量依然保留在了内存中。

### 3. 价格
无服务端架构（server-less）并不意味着低成本（cost-less）。对，绝对不会是低成本：总有一天你要为你的懒惰付出代价。然后，我们在收到了Firebase的邮件。我们真的付出了代价。

>（译：抱歉，我翻译商业邮件的感觉）Firebase的收费方案是有用量限制的。超过了套餐限制就要额外付费。你的账户crip已经超过了套餐限制。但是由于我们这边的事物，你并没有因此而被收取额外费用。我们从这周起会开始正常收费，也即是说从你的账户里面收取因超出流量产生的费用。
>

Firebase我日你大爷！（Thank you for the mail, Firebase!
）

用无服务器架构前必须要三思。分分钟你花100刀在Firebase上做的事情，花5刀在DigitalOcean上就能做到。

自己写服务端代码更好。你能让自己的产品获得更大的可维护性和生产力。还有一个高效的代码库。

### 4. Firebase分段加载数据

举个例子，你要建一个将Slack（一个高效的在线聊天应用）一样的应用，用Firebase的话，在app加载时，你就需要将全部的聊天室数据下载app里。

有人说，可以优化啊，为什么不分段加载数据？然而Firebase就是不可以。你获取不了数组长度，也不能将有序数组分页加载。

### 5. 数据不一致可能会出问题
Firebase支持离线操作，原理与`git commit`相似。问题在于，如果你的客户端先离线然后又在线，然后这过程中对同一个数据进行了操作（比如，同时修改了一个记事本），最后，你就有可能会遇到不一致性问题。原理与git merge是产生冲突差不多。

### 6. 数据迁移问题
如果用上了Firebase，你就不能像操作SQL数据库（ORM数据库或者ODM数据库）那样去迁移数据。

这就是说，你需要写这样的代码
```
if (user && user.new_subdocument && user.new_subdocument.new_property) {  
  // Do stuff.
}
```
结果是，到处需要安全判断逻辑

### 7. 无法理清的逻辑关系

处理NoSQL的关联（relation）很难，但是处理Firebase的关联简直是痛苦的。

举个例子，一个user属于一个team，一个team有很多users

User：
```
{
    name : "John Doe",
    team_ids : [...]
}
```

Team:
```
{
	name: "Acem Inc",
	user_ids: [...]
}
```

这就是说，你的`User`表需要监控`team_ids`属性，




