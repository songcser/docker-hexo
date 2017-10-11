---
title: Task并发
date: 2017-09-26 22:23:40
tags:
---

Task是与事件循环交互的主要方式之一。 Task包装协程并跟踪它的完成。 Tasks是Future的子类，所以其他协程等待他们并且每一个都会获得task完成时返回的结果。

## 启动Task

使用create_task()创建Task实例启动任务。task将做为事件循环并发操作的一部分运行，只要loop正在运行，并且协程  没有返回。

```
# asyncio_create_task.py

import asyncio

async def task_func():
    print('in task_func')
    return 'the result'

async def main(loop):
    print('createing task')
    task = loop.create_task(task_func())
    print('waiting for {!r}'.format(task))
    return_value = await task
    print('task completed {!r}'.format(task))
    print('return value: {!r}'.format(return_value))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

这个例子中main()函数退出之前需要等待task返回结果。

```
# python3 asyncio_create_task.py

creating task
waiting for <Task pending coro=<task_func() running at asyncio_create_task.py:12>>
in task_func
task completed <Task finished coro=<task_func() done, defined at asyncio_create_task.py:12> result='the result'>
return value: 'the result'
```

## 取消Task

通过create_task()可以返回Task对象，就有可能取消task的操作在它完成之前。

```
# asyncio_cancel_task.py

import asyncio

async def task_func():
    print('in task_func')
    return 'the result'

async def main(loop):
    print('createing task')
    task = loop.create_task(task_func())

    print('canceling task')
    task.cancel()

    print('canceled task {!r}'.format(task))
    try:
        await task
    except asyncio.CancelledError:
        print('caught error from canceled task')
    else:
        print('task result: {!r}'.format(task.result()))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

这个例子中在事件循环开始之前创建然后取消task。会从run_until_complete()函数中抛出CancelledError异常。

```
# python3 asyncio_cancel_task.py

creating task
canceling task
canceled task <Task cancelling coro=<task_func() running at asyncio_cancel_task.py:12>>
caught error from canceled task
```

如果task在等待其他并发操作时被取消，它会在它等待的地方抛出CancelledError异常来通知task被取消。

```
# asyncio_cancel_task2.py

import asyncio

async def task_func():
    print('in task_func, sleeping')
    try:
        await asyncio.sleep(1)
    except asyncio.CancelledError:
        print('task_func was canceled')
        raise
    return 'the result'

def task_canceller(t):
    print('in task_canceller')
    t.cancel()
    print('canceled the task')

async def main(loop):
    print('createing task')
    task = loop.create_task(task_func())
    loop.call_soon(task_canceller, task)
    try:
        await task
    except asyncio.CancelledError:
        print('main() also sees task as canceled')


event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

如果需要，截获异常能够提供机会来清理已经完成的任务。

```
$ python3 asyncio_cancel_task2.py

creating task
in task_func, sleeping
in task_canceller
canceled the task
task_func was canceled
main() also sees task as canceled
```

## 从协程生成Task

ensure_future()函数返回Task绑定到协程上。Task实例可以通过其他代码等待执行，而不知道原始协程被构造或调用。

```
# asyncio_ensure_future.py

import asyncio

async def wrapped():
    print('wrapped')
    return 'result'

async def inner(task):
    print('inner: starting')
    print('inner: waiting for {!r}'.format(task))
    result = await task
    print('inner: task returned {!r}'.format(result))


async def starter():
    print('starter: creating task')
    task = asyncio.ensure_future(wrapped())
    print('starter: waiting for inner')
    await inner(task)
    print('starter: inner returned')

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    result = event_loop.run_until_complete(starter())
finally:
    event_loop.close()
```

记住ensure_future()函数生成的协程不会执行,直到使用await允许它执行。

``` bash
$ python3 asyncio_ensure_future.py

entering event loop
starter: creating task
starter: waiting for inner
inner: starting
inner: waiting for <Task pending coro=<wrapped() running at asyncio_ensure_future.py:12>>
wrapped
inner: task returned 'result'
starter: inner returned
```

[原文链接](https://pymotw.com/3/asyncio/tasks.html)
