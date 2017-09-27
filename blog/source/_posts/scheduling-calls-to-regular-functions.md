---
title: 常规函数调用
date: 2017-09-26 10:48:23
tags:
---

除了管理协程和IO回调，asyncio事件循环可以根据loop中保留的定时器来调度常规函数。

## 调度回调函数"Soon"

如果回调的定时器不重要，call_soon()方法在loop的下次迭代中进行调度。方法被调用时可以将一些额外的位置参数传递给回调。可以使用functools模块的partial()方法将关键字参数传给回调。

```
# asyncio_call_soon.py

import asyncio
import functools

def callback(arg, *, kwarg='default'):
    print('callback invoked with {} and {}'.format(arg, kwarg))

async def main(loop):
    print('registering callbacks')
    loop.call_soon(callback, 1)
    wrapped = functools.partial(callback, kwarg='not default')
    loop.call_soon(wrapped, 2)

    await asyncio.sleep(0.1)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_untile_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```
回调函数按顺序调度执行。

```
# python3 asyncio_call_soon.py

entering event loop
registering callbacks
callback invoked with 1 and defalut
callback invoked with 2 and not defalut
closing event loop
```

## 延迟调度回调

使用call_later()推迟一段时间调用回调。第一个参数是延迟的秒数，第二个参数是回调函数。

```
# asyncio_call_later.py

import asyncio

def callback(n):
    print('callback {} invoked'.format(n))

async def main(loop):
    print('registering callbacks')
    loop.call_later(0.2, callback, 1)
    loop.call_later(0.1, callback, 2)
    loop.call_soon(callback, 3)

    await asyncio.sleep(0.4)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_untile_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```

在这个例子中，一些回调函数在不同的时间使用不同的参数被调用。使用call_soon()调用的最后的例子，带着参数3的回调函数要在时间延迟调度例子的前面获取结果，显然"soon"通常暗示最小的延迟。

```
# python3 asyncio_call_later.py

entering event loop
registering callbacks
callback 3 invoked
callback 2 invoked
callback 1 invoked
closing event loop
```

## 在具体时间调度回调函数

也可以在具体时间调用回调函数。loop使用monotonic clock而不是wall-clock time，以确保"now"的值不会后退。选择调度回调函数的时间需要从内部状态开始，要使用loop的time()函数。

```
# asyncio_call_at.py

import asyncio
import time

def callback(n, loop):
    print('callback {} invoked at {}'.format(n, loop.time()))

async def main(loop):
    now = loop.time()
    print('clock time: {}'.format(time.time()))
    print('loop time: {}'.format(now))

    print('registering callbacks')
    loop.call_at(now + 0.2, callback, 1, loop)
    loop.call_at(now + 0.1, callback, 2, loop)
    loop.call_soon(callback, 3, loop)

    await asyncio.sleep(1)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_untile_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```

注意: 时间按照loop的时间而不是time.time()返回的值。

```
# python3 asyncio_call_at.py

entering event loop
clock time: 1479050248.66192
loop time: 1008846.13856885
registering callbacks
callback 3 invoked at 1008846.13867956
callback 2 invoked at 1008846.239931555
callback 1 invoked at 1008846.343480996
closing event loop
```
