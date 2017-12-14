---
title: asyncio之异步生成结果
date: 2017-09-26 18:01:09
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

Future代表任务的结果还没有生成。事件循环能够监控Future对象的状态来标示完成，并且允许应用的一部分等待另一部分完成任务。

## 等待Future

Future的行为和协程一样，因此一些有用的等待协程的方法也可以用来等待Future的完成。这个例子将future传递给事件循环的run_until_complete()方法。

```python
# asyncio_future_event_loop.py

import asyncio

def mark_done(future, result):
    print('setting future result to {!r}'.format(result))
    future.set_result(result)

event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()

    print('acheduling mark_done')
    event_loop.call_soon(mark_done, all_done, 'the result')

    print('entering event loop')
    result = event_loop.run_untile_complete(all_done)
    print('returned result: {!r}'.format(result))
finally:
    print('closing event loop')
    event_loop.close()

print('future result: {!r}'.format(all_done.result()))
```

当set_result()方法被调用时Future的state改变了，并且Future实例保存结果以便最后返回。

```
$ python3 asyncio_future_event_loop.py

scheduling mark_done
entering event loop
setting future result to 'the result'
returned result: 'the result'
closing event loop
future result: 'the result'
```

Future也可以使用await关键字，如下所示。

```python
# asycnio_future_await.py

import asyncio

def mark_done(future, result):
    print('setting future result to {!r}'.format(result))
    future.set_result(result)

async def main(loop):
    all_done = asyncio.Future()

    print('scheduling mark_done')
    loop.call_soon(mark_done, all_done, 'the result')

    result = await all_done
    print('returned result: {!r}'.format(result))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_untile_complete(main(event_loop))
finally:
    event_loop.close()
```

Future的结果通过await返回，因此经常有可能普通协程函数和Future实例使用相同的代码。

```
$ python3 asyncio_future_await.py

scheduling mark_done
setting future result to 'the result'
returned result: 'the result'
```

## Future回调函数

除了像协程一样工作，Future也可以在执行结束之后调用回调函数。回调函数按照注册的顺序进行调用。

```python
# asyncio_future_callback.py

import asyncio
import functools

def callback(future, n):
    print('{}: future done: {}'.format(n, future.result()))

async def register_callbacks(all_done):
    print('registering callbacks on future')
    all_done.add_done_callback(functools.partial(callback, n=1))
    all_done.add_done_callback(functools.partial(callback, n=2))

async def main(all_done):
    await register_callbacks(all_done)
    print('setting result of future')
    all_done.set_result('the result')

event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()
    event_loop.run_untile_complete(main(all_done))
finally:
    event_loop.close()
```

回调函数要求Future实例为第一个参数，传给回调的额外参数使用functools.partial()进行包裹

```
# python3 asyncio_future_callback.py

registering callbacks on future
setting result of future
1: future done: the result
2: future done: the result
```

[原文链接](https://pymotw.com/3/asyncio/futures.html)
