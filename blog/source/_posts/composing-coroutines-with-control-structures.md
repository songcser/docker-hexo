---
title: asyncio之协程控制结构
date: 2017-10-10 01:13:17
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

使用语言内置的关键字await,一些协程之间的线性控制流是很容易进行管理的。更加复杂的结构使用asyncio的工具也可以做到允许一个协程等待其他协程的并发完成。

## 等待多个协程运行

经常会分割一个操作为多个部分分离执行。例如，下载多个远程资源或者查询远程API。在这种情况下，执行的顺序是无关紧要的并且可以有任意数量的操作，wait()方法能够停止一个协程等待其他协程完成后台操作。

``` python
# asyncio_wait.py

import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    await asyncio.sleep(0.1 * i)
    print('done with phase {}'.format(i))
    return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [
        phase(i)
        for i in range(num_phases)
    ]
    print('waiting for phases to complete')
    completed, pending = await asyncio.wait(phases)
    results = [t.result() for t in completed]
    print('results: {!r}'.format(results))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

在里面wait()使用set来装载Task实例。Task启动和结束都是在不可预料的顺序中，wait()函数的返回值是tuple类型，包含了完成和未完成的两个set集。

``` bash
$ python3 asyncio_wait.py

starting main
waiting for phases to complete
in phase 0
in phase 1
in phase 2
done with phase 0
done with phase 1
done with phase 2
results: ['phase 1 result', 'phase 0 result', 'phase 2 result']
```

如果wait()函数加上超时时间，将只剩下未完成的操作。

``` python
# asyncio_wait_timeout.py

import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    try:
        await asyncio.sleep(0.1*i)
    except asyncio.CancelledError:
        print('phase {} canceled'.format(i))
        raise
    else:
        print('done with phase {}'.format(i))
        return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [
        phase(i)
        for i in range(num_phases)
    ]
    print('waiting 0.1 for phases to complete')
    completed, pending = await asyncio.wait(phases, timeout=0.1)
    print('{} completed and {} pending'.format(
        len(completed), len(pending)
    ))
    # Cancel remaining tasks so they do not generate errors
    # as we exit without finishing them.
    if pending:
        print('canceling tasks')
        for t in pending:
            t.cancel()
    print('exiting main')

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

剩下的后台任务应该被取消或者等待完成。当事件循环继续执行时挂起的任务将继续执行，如果全部的操作被终止了，这可能被认为是不可取的。在进程结束时还有挂起的任务将报警告信息。

``` bash
# python3 asyncio_wait_timeout.py

starting main
waiting 0.1 for phases to complete
in phase 1
in phase 0
in phase 2
done with phase 0
1 completed and 2 pending
cancelling tasks
exiting main
phase 1 cancelled
phase 2 cancelled
```

## 获取协程的值

如果执行语句已经定义好，只关心它们的结果，使用gather()函数来等待多个操作可能更加有用。

``` python
# asyncio_gather.py

import asyncio

async def phase1():
    print('in phase1')
    await asyncio.sleep(2)
    print('done with phase1')
    return 'phase1 result'

async def phase2():
    print('in phase2')
    await asyncio.sleep(1)
    print('done with phase2')
    return 'pahse2 result'

async def main():
    print('starting main')
    print('waiting for phases to complete')
    results = await asyncio.gather(
        phase1(),
        phase2(),
    )
    print('results: {!r}'.format(results))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main())
finally:
    event_loop.close()
```

由gather创建的task没有暴露出来，因此task不能被取消。返回值在一个列表里和传给gather()函数的参数的顺序相同，不管后台任务执行完成的顺序。

``` bash
$ python3 asyncio_gather.py

starting main
waiting for phases to complete
in phase2
in phase1
done with phase2
done with phase1
results: ['phase1 result', 'phase2 result']
```

## 后台执行完成处理

as_completed()是一个生成器，管理一个链表里的协程，在协程执行完之后立即消费它的结果。和wait()一样，as_completed()是不保证顺序的，但是没有必要等待所有的后台执行完成再执行其他的行为。

``` python
# asyncio_as_completed.py

import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    await asyncio.sleep(0.5 - (0.1 * i))
    print('done with phase {}'.format(i))
    return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [
        phase(i)
        for i in range(num_phases)
    ]
    print('waiting for phases to complete')
    results = []
    for next_to_complete in asyncio.as_completed(phases):
        answer = await next_to_complete
        print('received answer {!r}'.format(answer))
        results.append(answer)
    print('results: {!r}'.format(results))
    return results

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```

这个例子中先开始的执行语句却是相反的顺序完成。当生成器被消费，loop使用await等待协程的结果。

``` bash
$ python3 asyncio_as_completed.py

starting main
waiting for phases to complete
in phase 0
in phase 2
in phase 1
done with phase 2
received answer 'phase 2 result'
done with phase 1
received answer 'phase 1 result'
done with phase 0
received answer 'phase 0 result'
results: ['phase 2 result', 'phase 1 result', 'phase 0 result']
```

[原文链接](https://pymotw.com/3/asyncio/control.html)
