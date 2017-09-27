---
title: 协程的多任务处理
date: 2017-09-25 10:52:24
tags: asyncio 异步 协程
---

协程是一种用于并发操作的语言结构。协程函数在被调用时创建协程对象，调用者可以使用协程的send()的方法来运行代码。协程通过await可以停止其他协程的执行。停止后，协程的状态是meaintained,在下次被唤醒时从这个地方继续运行。

## 开始一个协程

有几种不同的方法使用asyncio事件循环启动协程。最简单的方式是使用run_until_complete()方法，直接执行协程。

```
# asyncio_coroutine.py

import asyncio

async def coroutine():
    print('in coroutine')

event_loop = asyncio.get_event_loop()
try:
    print('starting coroutine')
    coro = coroutine()
    print('entering event loop')
    event_loop.run_untile_complete(coro)
finally:
    print('closing event loop')
    event_loop.close()
```

首先获取事件循环的引用，使用默认的loop或者指定特定的loop类。在这个例子中，使用类默认的loop。run_untile_complete()方法使用协程对象启动loop，当协程退出返回时停止loop。

```
# python3 asyncio_coroutine.py

starting coroutine
entering event loop
in coroutine
closing event loop
```

## 协程的返回值

协程的返回值是从它开始启动并且等待运行结束的代码处返回的。

```
# asyncio_coroutine_return.py

import asyncio

async def coroutine():
    print('in coroutine')
    return 'result'

event_loop = asyncio.get_event_loop()
try:
    retrun_value = event_loop.run_untile_complete(
        coroutine()
    )
    print('it returned: {!r}'.format(return_value))
finally:
    event_loop.close()
```
在这个例子中，run_untile_complete()方法返回了协程的值。

```
# python3 asyncio_coroutine_return.py

in coroutine
it returned: 'result'
```

## 协程链

一个协程可以启动另一个协程并且等待值的返回。 这使得更容易将任务分解成可重用的部分。下面的例子中有两个段落是按顺序执行的，但可以和其他操作并行运行。

```
# asyncio_coroutine_chain.py

import asyncio

async def outer():
    print('in outer')
    print('waiting for result1')
    result1 = await phase1()
    print('waiting for result2')
    result2 = await phase2(result1)
    return (result1, result2)

async def phase1():
    print('in phase1')
    return 'result1'

async def phase2(arg):
    print('in phase2')
    return 'result2 derived from {}'.format(arg)

event_loop = asyncio.get_event_loop()
try:
    return_value =event_loop.run_untile_complete(outer())
    print('return value: {!r}'.format(return_value))
finally:
    event_loop.close()
```
使用await关键字而不是添加新的协程到loop，因为被loop管理的控制流已经进入协程内部，所以不需要告诉loop去管理新的协程。

```
# python3 asyncio_coroutine_chain.py

in outer
waiting for result1
in phase1
waiting for result2
in phase2
return value: ('result', 'result2 derived from result1')
```

## 协程代替生成器

协程函数是asyncio的关键组件。它提供了语言级别的特性，可以停止程序执行的一部分，并且保存调用的状态，和最后重新进入的状态，这些都是并发框架具备的重要功能。

Python 3.5加入了新的语言特性，使用async def来定义原生的协程和使用await获取控制权。asyncio的例子都使用了新的特性。较早的Python3版本使用生成器函数，asyncio.coroutine()装饰器和yield from完成了相同的功能。

```
# asyncio_generator.py

import asyncio

@asyncio.coroutine
def outer():
    print('in outer')
    print('waiting for result1')
    result1 = yield from phase1()
    print('waiting for result2')
    result2 = yield from phase2(result1)
    return (result1, result2)

@asyncio.coroutine
def phase1():
    print('in phase1')
    return 'result1'

@asyncio.coroutine
def phase2(arg):
    print('in phase2')
    return 'result2 derived from {}'.format(arg)

event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_untile_complete(outer())
    print('return value: {!r}'.format(return_value))
finally:
    event_loop.close()
```

上面的例子使用生成器代替原生协程模仿asyncio_coroutine_chain.py

```
# python3 asyncio_generator.py

in outer
waiting for result1
in phase1
waiting for result2
in phase2
return value: ('result1', 'result2 derived from result1')
```

[原文链接](https://pymotw.com/3/asyncio/coroutines.html)
