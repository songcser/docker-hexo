---
title: asyncio之接收Unix信号
date: 2017-11-06 01:12:54
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

通常Unix系统事件通知需要中断应用来触发执行。当使用asyncio时，信号处理回调会与事件循环管理的其他协程的回调交错进行。这导致中断的功能更少，因此需要提供安全防护来清理不完整的操作。

信号处理必须是常规的函数调用，不是协程。

```python
# asyncio_signal.py

import asyncio
import functools
import os
import signal

def signal_handler(name):
    print('signal_handler({!r})'.format(name))
```

使用add_signal_handler()注册信号处理。第一个参数是信号，第二个参数是回调。回调函数是没有参数的，所以如果函数需要参数可以使用functools.partial()。

```python
event_loop = asyncio.get_event_loop()

event_loop.add_signal_handler(
    signal.SIGHUP,
    functools.partial(signal_handler, name="SIGHUP"),
)
event_loop.add_signal_handler(
    signal.SIGUSR1,
    functools.partial(signal_handler, name='SIGUSR1'),
)
event_loop.add_signal_handler(
    signal.SIGINT,
    functools.partial(signal_handler, name='SIGINT'),
)
```

这个示例使用协程通过os.kill()发送信号给它自己。信号发送之后，协程让出控制来让处理程序运行。在一般的应用中，有更多的地方是应用代码返还给事件循环而不是像这样人为的返还。

```python
async def send_signals():
    pid = os.getpid()
    print('starting send_signals for {}'.format(pid))

    for name in ['SIGHUP', 'SIGHUP', 'SIGUSR1', 'SIGINT']:
        print('sending {}'.format(name))
        os.kill(pid, getattr(signal, name))
        # Yield control to allow the signal handler to run,
        # since the signal does not interrupt the program
        # flow otherwise.
        print('yielding control')
        await asyncio.sleep(0.01)
    return
```

主程序运行send_signals()直到所有的信号发送完。

```python
try:
    event_loop.run_until_complete(send_signals())
finally:
    event_loop.close()
```

输出显示当发送信号之后send_signals()让出控制权，处理函数被调用。

```
$ python3 asyncio_signal.py

starting send_signals for 21772
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGUSR1
yielding control
yield_handler('SIGUSR1')
sending SIGINT
yielding control
signal_handler('SIGINT')
```

>>> [signal](https://pymotw.com/3/signal/index.html#module-signal) -  Receive notification of asynchronous system events

[原文链接](https://pymotw.com/3/asyncio/unix_signals.html)
