---
title: 协程和线程进程组合
date: 2017-11-13 07:32:01
tags:
---

许多已经存在的库还没有使用原生的asyncio。可能造成阻塞或者依赖模块不可用的并发特性。还是有可能在基于asyncio的应用中使用这些库，通过使用concurrent.futures的executor可以在一个分离的线程或者进程中运行代码。

## 线程

事件循环的run_in_executor()方法生成executor实例，一个常规的函数调用，并且传递一些参数。方法返回Future对象，常常用来等待函数完成任务和返回值。如果没有传入执行executor，就会创建ThreadPoolExecutor实例。这个例子明确的创建一个executor，来限制可获得工作线程的数量。

ThreadPoolExecutor启动工作线程，然后在线程中调用每一个提供的函数。这个例子显示了如何组合run_in_executor()和wait()方法在一个协程中，当阻塞函数在一个分离的线程中运行时将控制权返还给事件循环，并且在阻塞函数完成时再唤醒。

```python
# asyncio_executor_thread.py

import asyncio
import concurrent.futures
import logging
import sys
import time

def blocks(n):
    log = logging.getLogger('blocks({})'.format(n))
    log.info('running')
    time.sleep(0.1)
    log.info('done')
    return n ** 2

async def run_blocking_tasks(executor):
    log = logging.getLogger('run_blocking_tasks')
    log.info('starting')

    log.info('creating executor tasks')
    loop = asyncio.get_event_loop()
    blocking_tasks = [
        loop.run_in_executor(executor, blocks, i)
        for i in range(6)
    ]
    log.info('waiting for executor tasks')
    completed, pending = await asyncio.wait(blocking_tasks)
    results = [t.result() for t in completed]
    log.info('results: {!r}'.format(results))

    log.info('exiting')

if __name__ == '__main__':
    # Configure logging to show the name of the thread
    # where the log message originates.
    logging.basicConfig(
        level=logging.INFO,
        format='%(threadName)10s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    # Create a limited thread pool
    executor = concurrent.futures.ThreadPoolExecutor(
        max_workers=3,
    )

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(
            run_blocking_tasks(executor)
        )
    finally:
        event_loop.close()
```

asyncio_executor_thread.py使用logging可以方便的表明哪一个线程和函数产生的每一条日志信息。由于在每一个blocks()函数中调用使用分离的logger,输出清晰的显示了重复使用相同的线程以不同的参数跳用函数的多个副本。

```
$ python3 asyncio_executor_thread.py

MainThread run_blocking_tasks: starting
MainThread run_blocking_tasks: creating executor tasks
    Thread-1        blocks(0): running
    Thread-2        blocks(1): running
    Thread-3        blocks(2): running
MainThread run_blocking_tasks: waiting for executor tasks
    Thread-1        blocks(0): done
    Thread-3        blocks(2): done
    Thread-1        blocks(3): running
    Thread-2        blocks(1): done
    Thread-3        blocks(4): running
    Thread-2        blocks(5): running
    Thread-1        blocks(3): done
    Thread-2        blocks(5): done
    Thread-3        blocks(4): done
MainThread run_blocking_tasks: results: [16, 4, 1, 0, 2, 25, 9]
MainThread run_blocking_tasks: exiting
```

## 进程

ProcessPoolExecutor工作在相同的方式，代替线程创建一些工作进程。使用分离的进程需要更多的系统资源，但是对于计算密集型操作可以合理的在每一个CPU运行单独的任务。

```python
# asyncio_executor_process.py

# changes from asyncio_executor_thread.py

if __name__ == '__main__':
    # Configure logging to show the id of the process
    # where the log message originates.
    logging.basicConfig(
        level=logging.INFO,
        format='PID % (process)5s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    # Create a limited process pool.
    executor = concurrent.futures.ProcessPoolExecutor(
        max_workers=3,
    )

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(
            run_blocking_tasks(executor)
        )
    finally:
        event_loop.close()
```

将线程换成进程唯一的改变是创建一个不同类型的executor。这个例子也改变了日志格式字符串，将线程名字替换成了进程ID，来表明任务事实上运行在单独的进程中。

```
$ python3 asyncio_executor_process.py

PID 16429 run_blocking_tasks: starting
PID 16429 run_blocking_tasks: creating executor tasks
PID 16429 run_blocking_tasks: waiting for executor tasks
PID 16430           blocks(0): running
PID 16431           blocks(1): running
PID 16332           blocks(2): running
PID 16430           blocks(0): done
PID 16432           blocks(2): done
PID 16431           blocks(1): done
PID 16430           blocks(3): running
PID 16432           blocks(4): running
PID 16431           blocks(5): running
PID 16431           blocks(5): done
PID 16432           blocks(4): done
PID 16430           blocks(3): done
PID 16429 run_blocking_tasks: results: [4, 0, 16, 1, 9, 25]
PID 16429 run_blocking_tasks: exiting
```

[原文链接](https://pymotw.com/3/asyncio/executors.html)
