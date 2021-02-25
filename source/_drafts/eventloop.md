---
title: eventloop
date: 2018-09-14 17:48:59
tags:
  - JS基础
---


#### 结论
```
   ┌───────────────────────┐
┌─>│        timers         │<————— 执行 setTimeout()、setInterval() 的回调
│  └──────────┬────────────┘
|             |<-- 执行所有 Next Tick Queue 以及 MicroTask Queue 的回调
│  ┌──────────┴────────────┐
│  │     pending callbacks │<————— 执行由上一个 Tick 延迟下来的 I/O 回调（待完善，可忽略）
│  └──────────┬────────────┘
|             |<-- 执行所有 Next Tick Queue 以及 MicroTask Queue 的回调
│  ┌──────────┴────────────┐
│  │     idle, prepare     │<————— 内部调用（可忽略）
│  └──────────┬────────────┘     
|             |<-- 执行所有 Next Tick Queue 以及 MicroTask Queue 的回调
|             |                   ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │ - (执行几乎所有的回调，除了 close callbacks 以及 timers 调度的回调和 setImmediate() 调度的回调，在恰当的时机将会阻塞在此阶段)
│  │         poll          │<─────┤  connections, │ 
│  └──────────┬────────────┘      │   data, etc.  │ 
│             |                   |               | 
|             |                   └───────────────┘
|             |<-- 执行所有 Next Tick Queue 以及 MicroTask Queue 的回调
|  ┌──────────┴────────────┐      
│  │        check          │<————— setImmediate() 的回调将会在这个阶段执行
│  └──────────┬────────────┘
|             |<-- 执行所有 Next Tick Queue 以及 MicroTask Queue 的回调
│  ┌──────────┴────────────┐
└──┤    close callbacks    │<————— socket.on('close', ...)
   └───────────────────────┘
```

#### 对比浏览器
想理解整个 loop 的过程，我们可以参照浏览器的 event loop，因为浏览器的比较简单，如下：
```
   ┌───────────────────────┐
┌─>│        timers         │<————— 执行一个 MacroTask Queue 的回调
│  └──────────┬────────────┘
|             |<-- 执行所有 MicroTask Queue 的回调
| ────────────┘
```
是不是相比之下非常简洁，就这么两种 task queue，简单的一笔！
用一句话总结浏览器的 event loop 就是：

>先执行一个 MacroTask，然后执行所有的 MicroTask；
再执行一个 MacroTask，然后执行所有的 MicroTask；
……
如此反复，无穷无尽……

**注**：可以把 script 标签中的初始同步代码视为一个初始的 MacroTask


#### 解析
其实nodejs与浏览器的区别，就是nodejs的 MacroTask 分好几种，而这好几种又有不同的 task queue，而不同的 task queue 又有顺序区别，而 MicroTask 是穿插在每一种【注意不是每一个！】MacroTask 之间的。

其实图中已经画的很明白：
setTimeout/setInterval 属于 timers 类型；
setImmediate 属于 check 类型；
socket 的 close 事件属于 close callbacks 类型；
其他 MacroTask 都属于 poll 类型。
process.nextTick 本质上属于 MicroTask，但是它先于所有其他 MicroTask 执行；
所有 MicroTask 的执行时机，是不同类型 MacroTask 切换的时候。
idle/prepare 仅供内部调用，我们可以忽略。
pending callbacks 不太常见，我们也可以忽略。

所以我们可以按照浏览器的经验得出一个结论：
>先执行所有类型为 timers 的 MacroTask，然后执行所有的 MicroTask（注意 NextTick 要优先哦）；
进入 poll 阶段，执行几乎所有 MacroTask，然后执行所有的 MicroTask；
再执行所有类型为 check 的 MacroTask，然后执行所有的 MicroTask；
再执行所有类型为 close callbacks 的 MacroTask，然后执行所有的 MicroTask；
至此，完成一个 Tick，回到 timers 阶段；
……
如此反复，无穷无尽……

为了验证这个结论，我们甚至可以举一个例子：
```
setTimeout(()=>{
    console.log('timer1')
    Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)

setTimeout(()=>{
    console.log('timer2')
    Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)
```
此代码在浏览器环境会输出什么呢？
```
timer1
promise1
timer2
promise2
```
![图解浏览器 event loop 过程](http://upload-images.jianshu.io/upload_images/2707400-2968b449856af912.gif?imageMogr2/auto-orient/strip)

但是 nodejs 会输出：
```
timer1
timer2
promise1
promise2
```
![图解 nodejs event loop 过程](http://upload-images.jianshu.io/upload_images/2707400-781ed56509d40758.gif?imageMogr2/auto-orient/strip)


如果你已经理解了上面的现象，那我们已经算基本了解 nodejs 的 event loop 了，但是其中还有一点细节

#### 细节一：setTimeout 与 setImmediate 的顺序
本来这不应该成为一个问题，因为在文首显而易见，timers 是在 check 之前的。
但事实上，Node 并不能保证 timers 在预设时间到了就会立即执行，因为 Node 对 timers 的过期检查不一定靠谱，它会受机器上其它运行程序影响，或者那个时间点主线程不空闲。比如下面的代码，setTimeout() 和 setImmediate() 都写在 Main 进程中，但它们的执行顺序是不确定的：
```
setTimeout(() => {
  console.log('timeout')
}, 0)

setImmediate(() => {
  console.log('immediate')
})
```
虽然 setTimeout 延时为 0，但是一般情况 Node 把 0 会设置为 1ms，所以，当 Node 准备 event loop 的时间大于 1ms 时，进入 timers 阶段时，setTimeout 已经到期，则会先执行 setTimeout；反之，若进入 timers 阶段用时小于 1ms，setTimeout 尚未到期，则会错过 timers 阶段，先进入 check 阶段，而先执行 setImmediate

**但有一种情况，它们两者的顺序是固定的：**
```
const fs = require('fs')

fs.readFile('test.txt', () => {
  console.log('readFile')
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```
和之前情况的区别在于，此时 setTimeout 和 setImmediate 是写在 I/O callbacks 中的，这意味着，我们处于 poll 阶段，然后是 check 阶段，所以这时无论 setTimeout 到期多么迅速，都会先执行 setImmediate。本质上是因为，我们从 poll 阶段开始执行，而非一个 Tick 的初始阶段

#### 细节二：poll 阶段
poll 阶段主要有两个功能：
* 获取新的 I/O 事件，并执行这些 I/O 的回调，之后适当的条件下 node 将阻塞在这里
* 当有 immediate 或已超时的 timers，执行它们的回调

poll 阶段用于获取并执行几乎所有 I/O 事件回调，是使得 node event loop 得以无限循环下去的重要阶段。所以它的首要任务就是同步执行所有 poll queue 中的所有 callbacks 直到 queue 被清空或者已执行的 callbacks 达到一定上限，然后结束 poll 阶段，接下来会有几种情况：
1. setImmediate 的 queue 不为空，则进入 check 阶段，然后是 close callbacks 阶段……
2. setImmediate 的 queue 为空，但是 timers 的 queue 不为空，则直接进入 timers 阶段，然后又来到 poll 阶段……
3. setImmediate 的 queue 为空，timers 的 queue 也为空，此时会阻塞在这里，因为无事可做，也确实没有循环下去的必要

#### 细节三：关于 pending callbacks 阶段
在很多文章中，将 pending callbacks 阶段都写作 I/O callbacks 阶段，并说在此阶段，执行了除 close callbacks、 timers、setImmediate以外的几乎所有的回调，也就是把 poll 阶段的工作与此阶段的工作混淆了。
在我阅读时，就曾产生过疑问，假如大部分回调是在 I/O callbacks 阶段执行的，那么 poll 阶段就没有理由阻塞，因为你并不能保证“无事可做”，你得去 I/O callbacks 阶段检查一下才知道嘛！
所以最终结合其他几篇文章以及对源码的分析，应该可以确定，I/O callbacks 更准确的叫做 pending callbacks，它所执行的回调是比较特殊的、且不需要关心的，而真正重要的、大部分回调所执行的阶段是在 poll 阶段。

关于 pending callbacks 有如下说法，可以作为参考
>查阅了[libuv 的文档](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)后发现，在 libuv 的 event loop 中，`I/O callbacks` 阶段会执行 `Pending callbacks`。绝大多数情况下，在 `poll` 阶段，所有的 I/O 回调都已经被执行。但是，在某些情况下，有一些回调会被延迟到下一次循环执行。也就是说，在 `I/O callbacks` 阶段执行的回调函数，是上一次事件循环中被延迟执行的回调函数。

>严格来说，i/o callbacks并不是处理文件i/o的callback 而是处理一些系统调用错误，比如网络 stream, pipe, tcp, udp通信的错误callback。[参考](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#i-o-callbacks) 因为，pending_queue的入列（queue_insert_tail）是通过一个叫 uv__io_feed 的api来调用的 而 uv__io_feed API是在tcp/udp/stream/pipe等相关API调用


[参考资料1](http://lynnelv.github.io/js-event-loop-nodejs)
[参考资料2](http://www.cnblogs.com/bingooo/p/6720540.html)

>注意，以上结论只针对node10及其之前版本，如果是node11版本则和浏览器的eventloop相似（作者还没仔细研究）

>另外，node8之前的版本也可能因为底层使用libuv的不同而产生不稳定的结果（坑爹啊）