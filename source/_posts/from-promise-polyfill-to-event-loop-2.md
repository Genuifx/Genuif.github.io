---
title: 从Promise到Event Loop（二）
description: "Javascript的宿主环境是怎么实现Event Loop的？ MicroTask和MacroTask的区别？nodejs的Tick和Phase了解一下~本文从promise-polyfill开始，深入介绍Event Loop的机制"
date: 2018-04-12 15:59:09
tags: Javascript
---

## 前言
在[上一章](../from-promise-polyfill-to-eventloop)我们提出了一个promise和setTimeout回调执行顺序的疑问，并且一起研究了promise-polyfill的源码实现。然而，最初的问题还是没有得到解答，同时引出了更多的关于异步时序的思考，为了真正解决这个问题，我们必须深入理解一些js实现底层的东西。

## 热身
我们从下面这张经典图解开始了解，（引用自[Robert](http://vimeo.com/96425312 "Robert")）
![js模型图](/images/eventloop/model.png "js模型图")

这是一张js实现的模型图，请注意我的措辞“js实现”；实际上ecmascript并没有规定任何Event Loop的实现内容（划重点），如果去搜v8的源码，你会发现上图右边的web api实现都找不到，不管是XMLHttpRequest还是setTimeout的实现，我们都找不到出处。

原因很简单，web api 或者说 BOM，DOM这些都不是ecma的语言规范。

图中的内容老生常谈了，几乎每个js教程关于Event Loop都会挂这个，但还是稍微介绍一下：

### 1. heap（堆）
用于分配对象，对象名实际上一个指向一堆无结构的内存区域（同C语言的结构体？），需要注意的是，当我们形成闭包的时候，实际上作用域链也是保存到堆了。

### 2. stack（栈）
函数调用栈，当调用一个函数的时候，我们把函数入栈，当函数执行完毕，则从当前栈中弹出函数，并且继续执行下一个函数。javascript是单线程的语言主要是提现在这里，因为一个“thread”只有一个调用栈，所以一个时间片下永远都只能“专心”处理一件事，调用栈同样有调用深度的限制，一个无限循环自我调用的函数会把栈搞爆。
![](/images/eventloop/1_tqkykdU69DFrxi82JOWLbQ.png)
> 引用自Medium

### 3. Event Loop
一个不断执行的循环，不断的从任务队列或者说消息队列读取需要执行的回调函数，当函数执行的时候，会执行入栈操作
所有例如`setTimeout(fn, 0)`这种操作并不能保证当前函数执行完了之后马上就运行，会有排队的延迟。zakas在[这篇文章](https://www.nczonline.net/blog/2011/09/19/script-yielding-with-setimmediate/)中还提到了timer的解析和耗电问题💯
Event Loop的实现一般看起来长这样：
```javascript
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```
ok，热身完毕，正餐开始。


## 并发模型和事件循环

### 以“Thread”为单位
我们常说javascript是单线程语言，首先需要明确一点的是上述的heap，stack和Event Loop都在一个“thread”下执行，对于chrome来说，每个window（页面窗口）就是一个“thread”，所以只有同一个“thread”下面的函数或者对象才能相互交流，而web worker和非同源的iframe都有自己独有的thread，相互之间并不能直接交流。

### Event Loop 依赖js宿主环境实现
ecma的spec并没有规定Event Loop应该怎么设计，这就决定了在不同的浏览器，不同的宿主环境（node，嵌入式）其Event Loop的实现不同，表现也不禁相同。

故而，promise与setTimeout的回调执行先后问题，实际上在不同浏览器不同版本运行结果都不相同。而由于没有规范规定实现，所有你不能说某个浏览器的是错误的，只能说哪种实现更为合理（当然现代浏览器的实现已经趋于一致。
![](/images/eventloop/WX20180327-233228.png)
> 同一份代码在不同浏览器的运行结果, 引用自[jake](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

### Event Loop 是类似队列（queue-like）的结构
对node来说，Event Loop不是一个单纯的队列，而是由一个个Phase和Tick组成，每个Phase和Tick又有自己的任务队列。详见下文。
对chrome来说，Event Loop又分为Task（又叫MarcoTask）和MicroTask，各有自己任务队列，详见下文。
其他浏览器的资料由于闭源，网上没有找到相关资料，但是从表现上看估计实现差不多。如果有相关的信息欢迎分享链接。
故而，Event Loop从宏观的角度看的确是一个依次执行任务队列，但从微观实现而言并非简单的队列便万事大吉。

### Ecma-262的Jobs和Job Queues
上面我们说过，ecma并没有Event Loop的规范，但是如果读过ecma-262的spec的同学就会发现，在[8-4](http://www.ecma-international.org/ecma-262/6.0/#sec-jobs-and-job-queues)有一个神奇的Jobs和Job Queues章节，被光速打脸了吗？不存在的

ecma的Jobs实际上为promise定义了标准实现。由于在es6，promise已经成为了语言标准之一，而ecma的引擎需要不依赖宿主实现promise，例如V8能够脱离浏览器单独运行，在没有浏览器webApi支持的情况下依然要支持promise实现。所以spec里面有这边规定并不奇怪。

所以Jobs Queues和Event Loop实际上是两个东西，那么当V8在chrome运行的时候，就会有一个问题，promise的回调应该在哪里运行？
如果在V8，Event Loop将无法知道捕获promise相关任务，二者并不能协调工作。
如果在Event Loop，必须覆盖了V8的相关实现。
于是Html5规范规定了宿主环境必须覆盖实现ecma的Job Queues，也是就是说在浏览器环境实际上promise的回调处理由Event Loop接手。
![](/images/eventloop/html5spec.jpg)

## Chrome的Task和MicroTask
Tasks一般被设计为浏览器能够介入其执行间歇，以便响应用户的操作，刷新UI，不管是页面的点击或者滚动事件，如果执行栈被阻塞的了，用户将会感觉到明显的卡顿。
![](/images/eventloop/1_MCt4ZC0dMVhJsgo1u6lpYw.jpeg)

MicroTask一般则为了快速响应回调处理，如果当前调用栈没有其他执行函数的话，就会尽快的按照顺序调用对应回调，所以如果当前上下文已经没有待处理函数，或者两个Task之间的间歇都会执行MicroTask。值得注意的是，如果当前执行的MicroTask又动态添加了一个MicroTask的话，新的MicroTask会在这个周期内马上执行~

### 类型
属于Task的的回调类型有： `setTimeout`，`XMLHttpRequest`， `IndexDB`，`事件IO`，`postMessage`
属于MicroTask的回调类型有：`Promise`，`MutationObserver`

### 执行规则
1. Task回调按照顺序一个个执行，每个Task之间允许浏览器进行UI渲染
2. MicroTask回调按照顺序一个个执行，并且有以下执行时机

关于chrome的Task和MicroTask主要从jake大大（google chrome技术支持）的[文章](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)中了解到，如果有大佬对这方面有更深入的了解的话，评论，rtx求指教！！！

## node的Libuv
同chrome，node也是使用V8引擎编译，解释ecma的语法。如果你认真看了前文，就知道node需要实现自己的EventLoop机制，以保证单线程的js能以异步方式处理并发事务。

node的Event Loop实现全部在[Libuv](http://nikhilm.github.io/uvbook/introduction.html#background)
> The node.js project began in 2009 as a JavaScript environment decoupled from the browser. Using Google’s V8 and Marc Lehmann’s libev, node.js combined a model of I/O – evented – with a language that was well suited to the style of programming; due to the way it had been shaped by browsers. As node.js grew in popularity, it was important to make it work on Windows, but libev ran only on Unix. The Windows equivalent of kernel event notification mechanisms like kqueue or (e)poll is IOCP. libuv was an abstraction around libev or IOCP depending on the platform, providing users an API based on libev. In the node-v0.9.0 version of libuv libev was removed.

一开始的node.js实现并不支持window环境，后来用户越来越多了，需要支持window环境，但是原来的libev又不支持，于是大佬们便自己整了一个libuv，实现了node自己的事件循环机制，抹平了底层差异，对Module提供统一的api。

当实例化一个node应用的时候，libuv会初始化一个线程池用于处理一些没有操作系统API支持的异步事件。也就是说libuv并非独立于js运行时，他和V8应该在一个线程下工作，异步IO处理也并非都在单独线程，libuv会优先使用系统提供API进行处理。

![](/images/eventloop/nodejs-EventLoop.jpg)

### Libuv的Event Loop
Libuv的Event Loop和chrome的类似，分为*`phase`*和*`tick`*，phase又有细分的种类，tick跟MicroTask类似，用于执行需要在当前运行时执行完毕后马上执行的事件回调，例如`promise`, `process.nextTick`

![](/images/eventloop/node-event-loop.png)

[nodejs官方示例](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)：
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘

libuv会循环的执行执行每个phase的相应队列，FIFO原则。

### 分类
#### Timer
所有通过setTimeout，setInterval调度的事件回调都在这里得到处理
### IO callback
绝大部分回调处理的时机，包括http请求引发的回调，文件读写IO回调
### IO Polling
给下一轮循环收集新的事件
### Immediate
通过setImmediate调度的事件回调处理
### close
处理所有的.on('close')事件回调

## 结论
综上所述：
1. 不同的宿主环境实现Event Loop不同
2. promise在不同浏览器不同版本执行顺序都有可能不一样，依赖实现
3. js是一个单线程语言，依赖异步模型达到并发效果
4. 代码层面不要让`Promise.resolve`和`setTimeout(fn, 0)`竞争执行速度

## 参考资料
ECMAScript 的 Job Queues 和 Event loop 有什么关系？ - flyingsoul的回答 - 知乎
https://www.zhihu.com/question/40063533/answer/271176956

[Ecma Jobs and Jobs Queues](http://www.ecma-international.org/ecma-262/6.0/#sec-jobs-and-job-queues)

[Mdn Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)

[What you should know to really understand the Node.js Event Loop](https://medium.com/the-node-js-collection/what-you-should-know-to-really-understand-the-node-js-event-loop-and-its-metrics-c4907b19da4c)

https://medium.com/@gaurav.pandvia/understanding-javascript-function-executions-tasks-event-loop-call-stack-more-part-1-5683dea1f5ec

https://stackoverflow.com/questions/10680601/nodejs-event-loop

https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/

http://nikhilm.github.io/uvbook/introduction.html#background



