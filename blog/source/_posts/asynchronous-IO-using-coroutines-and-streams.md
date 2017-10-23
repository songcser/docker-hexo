---
title: 协程异步IO流
date: 2017-10-20 01:05:15
tags:
---

这节练习两个例程实现简单的服务端和客户端的替换版本。使用协程和asyncio流API代替protocol和transport抽象类。这节操作较低级别的抽象层而不是以前讨论的Protocol API，但是产生的事件是相似的。

## 服务端

首先导入需要的模块asyncio和logging,然后创建事件循环对象。

```python
# asyncio_echo_server_coroutine.py

import asyncio
import logging
import sys

SERVER_ADDRESS = ('localhost', 10000)
logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(messages)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```

定义协程处理连接。每次客户端连接，一个新的协程实例将被调用以便功能代码同一时间只连接一个客户端。Python的语言运行时管理每个协程实例的状态，所以应用代码不需要管理额外的数据结构来跟踪不同的客户端。

协程的参数是StreamReader和StreamWriter实例来连接新的连接。和Transport一样，客户端的地址能够通过writer的方法get_extra_info()方法获得。

```python
async def echo(reader, writer):
    address = writer.get_extra_info('peername')
    log = logging.getLogger('echo_{}_{}'.format(*address))
    log.debug('connection accepted')
```

虽然当连接建立时协程被调用，但是可能没有数据读取。当读取时避免阻塞，协程使用await关键字来调用read()方法，允许事件循环继续处理其他的任务直到数据读取。

```python
    while True:
        data = await reader.read(128)
```

如果客户端发送数据，将会在await中返回数据并且把数据返还给客户端通过writer。多次调用write()经常会缓存输出的数据，然后drain()方法来刷新数据。由于刷新网络IO会阻塞，再使用await关键字将控制权返还给事件循环，监控着写socket和当可能发送更多数据时调用writer。

```python
        if data:
            log.debug('received {!r}'.format(data))
            writer.write(data)
            await writer.drain()
            log.debug('sent {!r}'.format(data))
```

如果客户端没有发送数据，read()方法返还空字串来表明连接关闭了。服务端需要关闭套接字返还给客户端，然后协程返还表明完成。

```python
        else:
            log.debug('closing')
            writer.close()
            return
```

两步来启动服务端。首先应用告诉事件循环使用协程和要监听的hostname和socket来创建新的服务端对象。start_server()方法本身是个协程，所以结果必须由事件循环处理才能实际启动服务器。完成协程会产生一个绑定到事件循环的asyncio.Server实例。

```python
# Create the server and let the loop finish the coroutine before 
# starting the real event loop.
factory = asyncio.start_server(echo, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} port {}'.format(*SERVER_ADDRESS))
```

事件循环需要运行为了处理事件和和客户端请求。对于长时间运行的服务，run_forever()方法是最简单的方法。当事件循环停止，无论通过应用代码或者通过进程信号，服务端能够关闭来正确清理socket，然后事件循环在程序退出之前被关闭以便来处理其他其他协程。

```python
# Enter the event loop permanently to handle all connections
try:
    event_loop.run_forever()
except KeyboradInterrupt:
    pass
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

## 客户端

使用协程构造客户端和构造服务端很相似。代码首先也是从导入需要的asyncio和logging模块开始，然后创建事件循环对象。

```python
# asyncio_echo_client_coroutine.py

import asyncio
import logging
import sys

MESSAGES = [
    b'This is the message. ',
    b'It will be sent '
    b'in parts.'
]
SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```

echo_client协程的参数表示服务端的地址和发送的信息。

```python
async def echo_client(address, messages):
```

当任务开始时协程被调用，但是还没有活跃的连接工作。所以首先要建立自己的连接。使用await关键字来避免阻塞其他活跃的连接当open_connection()协程运行时。

```python
    log = logging.getLogger('echo_client')

    log.debug('connection to {} port {}'.format(*address))
    reader, writer = await asyncio.open_connection(*address)
```

open_connection()协程返回StreamReader和StreamWriter实例来连接新的套接字。下一步是使用writer来发送数据给服务端。在服务端，writer将缓存输出的数据直到socket准备好或者使用drain()方法刷新结果。由于刷新网络IO会阻塞，再使用await关键字将控制权返还给事件循环，它监控着写入套接字并且调用writer当有可能发送更多的数据。

```python
    # This could be writer.writelines() except that
    # would make it harder to show each part of the message
    # being sent.
    for msg in messages:
        writer.write(msg)
        log.debug('sending {!r}'.format(msg))
    if writer.can_write_eof():
        writer.write_eof()
    await writer.drain()
```

接下来，客户端通过尝试读取数据来寻找服务端的响应直到没有数据进行读取。为了避免在单个read()调用上阻塞，await返还控制权给事件循环。如果服务端发送数据，记录日志。如果服务端没有发送数据，read()方法返还空字串来表明连接关闭了。客户端需要关闭套接字发送给服务端然后返回表明已经完成。

```python
    log.debug('waiting for response')
    while True:
        data = await reader.read(128)
        if data:
            log.debug('received {!r}'.format(data))
        else:
            log.debug('closing')
            writer.close()
            return
```

要启动客户端，将使用协程调用事件循环创建客户端。使用run_until_complete()方法避免在客户端程序中产生死循环。不像protocol的例子，当协程完成时不需要向之后单独发送信号，因为echo_client()包含所有的客户端处理逻辑，并且在收到响应和关闭服务器连接之前不会返回。

```python
try:
    event_loop.run_until_complete(
        echo_client(SERVER_ADDRESS, MESSAGES)
    )
finally:
    log.debug('closing event loop')
    event_loop.close()
```

## 输出

在一个窗口启动服务器，在另一个窗口启动客户端产生下面输出。

```
$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent in parts.'
echo_client: closing
main: closing event loop

$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent in parts.'
echo_client: closing
main: closing event loop

$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent '
echo_client: received b'in parts.'
echo_client: closing
main: closing event loop
```

尽管客户端一直单独的发送数据，客户端首先运行的两次，服务端接收了大量的信息并且输出返回给客户端。这些结果在后续运行中有所不同，这取决于网络的繁忙程度以及所有数据准备之前释放刷新网络缓冲区。

```
$ python3 asyncio_echo_server_coroutine.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
echo_::1_64624: connection accepted
echo_::1_64624: received b'This is the message. It will be sent in parts.'
echo_::1_64624: sent b'This is the message. It will be sent in parts.'
echo_::1_64624: closing

echo_::1_64626: connection accepted
echo_::1_64626: received b'This is the message. It will be sent in parts.'
echo_::1_64626: sent b'This is the message. It will be sent in parts.'
echo_::1_64626: closing

echo_::1_64627: connection accepted
echo_::1_64627: received b'This is the message. It will be sent '
echo_::1_64627: sent b'This is the message. It will be sent '
echo_::1_64627: received b'in parts.'
echo_::1_64627: sent b'in parts.'
echo_::1_64627: closing
```

[原文链接](https://pymotw.com/3/asyncio/io_coroutine.html)
