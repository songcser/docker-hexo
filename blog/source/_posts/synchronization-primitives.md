---
title: 同步原语
date: 2017-10-11 02:00:55
tags:
---

尽管asyncio应用经常是单线程进程运行，但是也同样可以同步应用构建。每一个coroutine或task可能在不可预测的顺序执行，基于延迟、IO中断和其他事件。为了支持安全同步，asyncio提供了和threading和multiprocessing模块相同的低水平的原始接口。

## 锁

锁可以保证访问共享资源。只有加锁的才能使用资源。多个试图获取锁的将会阻塞住以便同一时间只有一个获取锁。

```python
# asyncio_lock.py

import asyncio
import functools

def unlock(lock):
    print('callback releasing lock')
    lock.release()

async def coro1(lock):
    print('coro1 waiting for the lock')
    with await lock:
        print('coro1 acquired lock')
    print('coro1 released lock')

async def coro2(lock):
    print('coro2 waiting for the lock')
    await lock
    try:
        print('coro2 acquired lock')
    finally:
        print('coro2 released lock')
        lock.release()

async def main(loop):
    # Create and acquire a shared lock.

    lock = asyncio.Lock()
    print('acquiring the lock before starting coroutines')
    await lock.acquire()
    print('lock acquired: {}'.format(lock.locked()))

    # Schedule a callback to unlock the lock.
    loop.call_later(0.1, functools.partial(unlock, lock))

    # Run the coroutines that want to use the lock.
    print('waiting for coroutines')
    await asyncio.wait([coro1(lock), coro2(lock)]),

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

在这个例子中，lock可以直接调用，使用await获取锁，使用release()方法来释放锁在coro2()方法中。在coro1中，可以使用with await关键字作为同步上下文管理。

```
$ python3 asyncio_lock.py

acquiring the lock before starting coroutines
lock acquired: True
waiting for coroutines
coro1 waiting for the lock
coro2 waiting for the lock
callback releasing lock
coro1 acquired lock
coro1 released lock
coro2 acquired lock
coro2 released lock
```

## 事件

asyncio.Event基于threading.Event, 经常被用来允许多个消费者等待相关联的通知发生，而不需要特定的值。

``` python
# asyncio_event.py

import asyncio
import functools

def set_event(event):
    print('setting event in callback')
    event.set()

async def coro1(event):
    print('coro1 waiting for event')
    await event.wait()
    print('coro1 triggered')

async def coro2(event):
    print('coro2 waiting for event')
    await event.wait()
    print('coro2 triggered')

async def main(loop):
    # Create a shared event
    event = sayncio.Event()
    print('event start state: {}'.format(event.is_set()))
    
    loop.call_later(
        0.1, functools.partial(set_event, event)
    )
    await asyncio.wait([coro1(event), coro2(event)])
    print('event end state: {}'.format(event.is_set()))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

就像锁一样，coro1()和coro2()都等待事件的发生。不同的是只要事件状态变化就都开始执行。

```
$ python3 asyncio_event.py

event start state: False
coro2 waiting for event
coro1 waiting for event
setting event in callback
coro2 triggered
coro1 triggered
event end state: True
```

## 条件变量

Condition工作方式和Event相似，但是不是通知所有的等待的协程，而是根据传给notify()的参数只是唤醒一部分等待者。

```python
# asyncio_condition.py

import asyncio

async def consumer(condition, n):
    with await condition:
        print('consumer {} if waiting'.format(n))
        await condition.wait()
        print('consumer {} triggered'.format(n))
    print('ending consumer {}'.format(n))

async def manipulate_condition(condition):
    print('starting manipulate_condition')

    # pause to let consumers start
    await asyncio.sleep(0.1)

    for i in range(1, 3):
        with await condition:
            print('notifying {} consumers'.format(i))
            condition.notify(n=i)
        await asyncio.sleep(0.1)

    with await condition:
        print('notifying remaining consumers')
        condition.notify_all()

    print('ending maipulate_condition')

async def main(loop):
    # Create a condition
    condition = asyncio.Condition()

    # Set up tasks watching the condition
    consumers = [
        consumer(condition, i)
        for i in range(5)
    ]

    # Schedule a task to manipulate the condition variable
    loop.create_task(manipulate_condition(condition))

    # Wait for the consumers to be done
    await asyncio.wait(consumers)

event_loop = asyncio.get_event_loop()
try:
    result = event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```

这个例子中启动了5个Condition的消费者。每一个都使用wait()方法等待通知以便继续进行。manipulate_condition()方法中通知一个消费者，然后两个，最后是所有剩下的消费者。

``` 
$ python3 asyncio_condition.py

starting manipulate_condition
consumer 3 is waiting
consumer 1 is waiting
consumer 2 is waiting
consumer 0 is waiting
consumer 4 is waiting
notifying 1 consumers
consumer 3 triggered
ending consumer 3
notifying 2 consumers
consumer 1 triggered
ending consumer 1
consumer 2 triggered
ending consumer 2
notifying remaining consumers
ending manipulate_condition
consumer 0 triggered
ending consumer 0
consumer 4 triggered
ending consumer 4
```

## 队列

asyncio.Queue提供了先进先出的协程数据结构，就像线程中的queue.Queue和进程中的multiprocessing.Queue。

``` python
# asyncio_queue.py

import asyncio

async def consumer(n, q):
    print('consumer {}: starting'.format(n))
    while True:
        print('consumer {}: waiting for item'.format(n))
        item = await q.get()
        print('consumer {}: has item {}'.format(n, item))
        if item is None:
            # None is the signal to stop
            q.task_done()
            break
        else:
            await asyncio.sleep(0.01 * item)
            q.task_done()
    print('consumer {}: ending'.format(n))

async def producer(q, num_workers):
    print('producer: starting')
    # Add some numbers to the queue to simulate jobs
    for i in range(num_workers * 3):
        await q.put(i)
        print('producter: added task {} to the queue'.format(i))
    # Add None entries in the queue
    # to signal the consumers to exit
    print('producer: add stop signals to the queue')
    for i in range(num_workers):
        await q.put(None)
    print('producer: waiting for queue to empty')
    await q.join()
    print('producer: ending')

async def main(loop, num_consumers):
    # Create the queue with a fixed size so the producer
    # will block until the consumers pull some items out.
    q = asyncio.Queue(maxsize=num_consumers)

    # Scheduled the consumer tasks
    consumers = [
        loop.create_task(consumer(i, q))
        for i in range(num_consumers)
    ]

    # Schedule the producer task.
    prod = loop.create_task(producer(q, num_consumers))

    # Wait for all of the coroutines to finish
    await asyncio.wait(consumers + [prod])

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, 2))
finally:
    event_loop.close()
```

使用put()添加对象或者使用get()移除对象都是异步操作，直到队列数量固定（阻塞除外）或者队列为空（阻塞调用获取对象）

``` 
$ python3 asyncio_queue.py

consumer 0: starting
consumer 0: waiting for item
consumer 1: starting
consumer 1: waiting for item
producer: starting
producer: added task 0 to the queue
producer: added task 1 to the queue
consumer 0: has item 0
consumer 1: has item 1
producer: added task 2 to the queue
producer: added task 3 to the queue
consumer 0: waiting for item
consumer 0: has item 2
producer: added task 4 to the queue
consumer 1: waiting for item
consumer 1: has item 3
producer: added task 5 to the queue
producer: adding stop signals to the queue
consumer 0: waiting for item
consumer 0: has item 4
consumer 1: waiting for item
consumer 1: has item 5
producer: waiting for queue to empty
consumer 0: waiting for item
consumer 0: has item None
consumer 0: ending
consumer 1: waiting for item
consumer 1: has item None
consumer 1: ending
producer: ending
```

[原文链接](https://pymotw.com/3/asyncio/synchronization.html)
