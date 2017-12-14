---
title: asyncio之调试
date: 2017-11-21 01:44:14
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

asyncio构建了一些有用的调试特性。

首先，事件循环在运行时使用logging来发送状态信息。在应用中启用日志就会获取到一些信息。其他的可以打开告诉事件循环来发送更多的调试信息。通过调用set_debug()传递boolean值来表明是否启动调试。

由于构建在asyncio上的应用程序对贪婪的协程很敏感而无法返回控制权，因此对于检查内置于事件循环中的慢回调是有帮助的。通过打开启用调试，并且通过将loop的slow_callback_duration的属性设置为发出警告的秒数来控制"slow"的定义。

最后，如果应用程序使用asyncio退出但没有清理一些协程或者其他的资源，这或许意味着有一个逻辑错误阻止了一些应用代码运行。在程序退出时，启用ResourceWarning警告会报告这些情况。

```python
# asyncio_debug.py

import argparse
import asyncio
import logging
import sys
import time
import warnings

parser = argparse.ArgumentParser('debugging asyncio')
parser.add_argument(
    '-v',
    dest='verbose',
    default=False,
    action='store_true',
)
args = parser.parse_args()

logging.basicConfig(
    level=logging.DEBUG,
    format='%(levelname)7s: %(message)s',
    stream=sys.stderr,
)
LOG = logging.getLogger('')

async def inner():
    LOG.info('inner starting')
    # Use a blocking sleep to simulate
    # doing work inside the function.
    time.sleep(0.1)
    LOG.info('inner completed')

async def outer(loop):
    LOG.info('outer starting')
    await asyncio.ensure_future(loop.create_task(inner()))
    LOG.info('outer  completed')


event_loop = asyncio.get_event_loop()
if args.verbose:
    LOG.info('enabling debugging')

    # Enable debugging
    event_loop.set_debug(True)

    # Make the threshold for "slow" tasks very very small for
    # illustration. The default is 0.1, or 100 milliseconds.
    event_loop.slow_callback_duration = 0.001

    # Report all mistakes managing asynchronous resources.
    warnings.simplefilter('always', ResourceWarning)

LOG.info('entering event loop')
event_loop.run_until_complete(outer(event_loop))
```

当没有启用调试运行时，所有都看起来是好的。

```
$ python3 asyncio_debug.py

DEBUG: Using selector: KqueueSelector
 INFO: entering event loop
 INFO: outer starting
 INFO: inner starting
 INFO: inner completed
 INFO: outer completed
```

打开调试将暴露一些问题，包括虽然inner()方法完成，但是由于设置了slow_callback_duration，将需要更多的时间，并且程序退出时事件循环未被正确关闭的事实。

```
$ python3 asyncio_debug.py -v
DEBUG: Using selector: KqueueSelector
    INFO: enabling debugging
    INFO: entering event loop
    INFO: outer starting
    INFO: inner starting
    INFO: inner completed
WARNING: Executing <Task finished coro=<inner() done, defined at asyncio_debug.py:34> result=None created at asyncio_debug.py:44>
took 0.102 seconds
    INFO: outer completed
.../lib/python3.5/async/base_events.py:429: ResourceWarning:unclosed event loop <_UnixSelectorEventLoop running=False closed=False debug=Trur>
    DEBUG: Close <_UnixSelectorEventLoop running=False closed=False debug=True>
```

[原文链接](https://pymotw.com/3/asyncio/debugging.html)
