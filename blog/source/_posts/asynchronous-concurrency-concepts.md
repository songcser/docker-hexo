---
title: asyncio之异步协程概念
date: 2017-09-24 09:42:10
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

> 最近在学习python3中新的异步模块asyncio, [PyMOTW-3](https://pymotw.com/3/)中对asyncio有详细的介绍，所以借此机会将相关的文章翻译出来，供大家一起学习。
> 由于英文水平有限，翻译难免有些错误，希望大家指正。

现在大多数并发模型是线性写入的，并且依赖于语言运行时或操作系统的底层线程或进程进行管理,来适当的改变程序运行时的上下文。而基于asyncio的应用程序是由应用代码处理上下文的切换，使用正确的技术取决于几个相关的概念。

asyncio框架是以事件循环为中心，是负责高效处理IO事件，系统事件和应用上下文切换的第一类对象。并且提供了几个循环实现，以有效地利用操作系统功能。虽然通常可以自动的选择合理的默认值，但应用程序也可以选择特定的事件循环。这在Windows下是有用的，例如，一些支持外部程序的循环类在进行网络IO时可以提高效率。

应用程序和事件循环之间的交互需要注册代码运行，在资源可用时，事件循环可以调用应用程序的代码。例如，网络服务器在打开socket时进行注册以便接收到输入事件。当有新的连接接入或者有数据读取时，事件循环将提醒服务端代码。应用程序代码在当前上下文中没有任务做的短期时间内会重新让出控制运行。例如，没有更多的数据从套接字中读取时，服务端将控制权返还给事件循环。

出让控制权给事件循环的机制依赖于Python的协程。协程是一种特殊的函数能放弃控制权给调用者但不会丢失状态。协程和生成器很相似，事实上在Python 3.5版本以前没有原生的协程对象支持时生成器被用作实现协程。asycnio提供了protocols和transports的基类抽象层使用回调编写代码代替直接写协程。在基类和协程模块这两种方法都通过重新进入事件循环来明确的改变上下文运行环境，代替了Python的线程实现中隐式的上下文切换。

数据结构Future代表了还没有完成的任务结果。事件循环会监控Future对象直到完成，应用程序的一部分代码允许等待另一部分任务完成。除了features，asyncio还有以前的其他同步操作例如锁（locks）和信号量（semaphores）。

Task是Future的子类，能够包装和管理协程的执行。Task在它获取到需要的资源时，会被事件循环调度运行，并且生成结果给其他协程消费。

## 文章目录

1. [协程的多任务处理](https://www.songcser.com/2017/09/25/cooperative-multitasking-with-coroutines/)
2. [常规函数调用](https://www.songcser.com/2017/09/26/scheduling-calls-to-regular-functions/)
3. [异步生成结果](https://www.songcser.com/2017/09/26/producing-results-asynchronously/)
4. [执行并发任务](https://www.songcser.com/2017/09/26/executing-tasks-concurrently/)
5. [协程控制结构](https://www.songcser.com/2017/10/10/composing-coroutines-with-control-structures/)
6. [同步原语](https://www.songcser.com/2017/10/11/synchronization-primitives/)
7. [抽象类Protocol异步IO](https://www.songcser.com/2017/10/13/asynchronous-IO-with-protocol-class-abstractions/)
8. [协程异步IO流](https://www.songcser.com/2017/10/20/asynchronous-IO-using-coroutines-and-streams/)
9. [使用SSL](https://www.songcser.com/2017/10/23/using-ssl/)
10. [域名服务交互](https://www.songcser.com/2017/10/25/interacting-with-domain-name-services/)
11. [子进程交互](https://www.songcser.com/2017/10/26/working-with-subprocesses/)
12. [接收Unix信号](https://www.songcser.com/2017/11/06/receiving-unix-signals/)
13. [协程和线程进程组合](https://www.songcser.com/2017/11/13/combining-coroutines-with-threads-and-processes/)
14. [调试](https://www.songcser.com/2017/11/21/debugging-with-asyncio/)


[原文链接](https://pymotw.com/3/asyncio/concepts.html)
