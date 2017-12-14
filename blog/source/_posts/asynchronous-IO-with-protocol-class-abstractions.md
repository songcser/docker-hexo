---
title: asyncio之抽象类Protocol异步IO
date: 2017-10-13 07:54:30
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

到目前为止，那些例子都避免混合并发和IO操作，以便每次都关注一个概念。当IO阻塞时进行上下文切换是asyncio一个主要的用例。
基于已经介绍的并发概念，这节验证两个例程，一个简单的服务端和客户端，和在socket和socketserver那节中的例子相似。客户端能够连接服务端，发送数据然后接收相同的数据。每一次遇到IO操作时，执行代码就会放弃控制给事件循环，允许其他的任务运行直到IO操作完成。

## 服务端

服务端首先导入asyncio和logger模块，然后创建事件循环对象。

```python
# asyncio_echo_server.protocol.py

import asyncio
import logging
import sys

SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```

定义一个asyncio.Protocol的子类处理客户端的连接。基于事件的服务端socket连接发生时，protocol对象的方法被调用。

```python
class EchoServer(asyncio.Protocol):
```

每一个新的客户端连接都要触发connection_made()方法调用。transport的参数是asyncio.Transport实例，它提供了使用socket的抽象异步IO。不同的连接类型提供了不同的transport接口，都使用相同的API。例如，有单独的transport类用于处理socket和用于处理子进程的管道。客户端的地址可以通过tansport特定实现的接口get_extra_info()方法获得。

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log = logging.getLogger(
            'EchoServer_{}_{}'.format(*self.address)
        )
        self.log.debug('connection accepted')
```

链接建立之后，客户端传来的数据通过调用protocol的data_received()方法获得。数据以字节类型传输，到达应用时以合适的方法进行解码。日志记录下结果，然后通过调用tansport.write()将response立即返回给客户端。
```python
    def data_received(self, data):
        self.log.debug('received {!r}'.format(data))
        self.transport.write(data)
        self.log.debug('sent {!r}'.format(data))
```

一些transport支持特殊的端到文件("EOF")。当遇到EOF，eof_received()方法被调用。在这种实现中，EOF被返回给客户端表明数据已经被接收。由于不是所有的transport明确支持EOF，所以protocol首先询问transport发送EFO是否是安全的。

```python
    def eof_received(self):
        self.log.debug('reveived EOF')
        if self.transport.can_write_eof():
            self.transport.write_eof()
```

当链接关闭时，无论正常的或者错误的结果，protocol的connection_lost()方法都被调用。如果发生错误，参数中包含一个相关的异常对象，否则是None

```python
    def connection_lost(self, error):
        if error:
            self.log.error('ERROR: {}'.format(error))
        else:
            self.log.debug('closing')
        super().connection_lost(error)
```
需要两步启动服务。首先应用程序调用调用事件循环,使用protocol类以及要监听的hostname和socket来创建一个新的server对象。create_server()方法是协程，所以返回结果必须通过事件循环产生，并且启动server。完成协程会产生绑定到事件循环上的asyncio.Server实例。

```python
# Create the server and let the loop finish the coroutine before starting the real event loop.
factory = event_loop.create_server(EchoServer, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} prot {}'.format(*SERVER_ADDRESS))
```
然后，需要运行事件循环才能处理事件和处理客户端请求。对于长时间运行的服务，run_forever()方法是最简单的方式。当事件循环停止， 无论通过应用代码或者通过发信号给进程，都可以关闭server并且正确清理socket，然后在程序退出之前关闭事件循环以完成处理其他的协程。

```python
# Enter the event loop permanently to handle all connections.
try:
    event_loop.run_forever()
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

## 客户端

使用protocol类创建客户端和创建服务端很相似。代码还是从导入asyncio和logging模块开始，然后创建事件循环对象。

```python
# asyncio_echo_client_protocol.py

import asyncio
import functools
import logging
import sys

MESSAGES = [
    b'This is the message. ',
    b'It will be sent',
    b'in parts. ',
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

protocol类的客户端定义了和服务端相同的方法，使用了不同的接口。构造类接收两个参数，即需要发送的消息列表,以及Futrue实例，用来从服务端接收响应以通知客户端已经完成一个任务循环。

```python
class EchoClient(asyncio.Protocol):
    
    def __init__(self, messages, future):
        super().__init__()
        self.messages = messages
        self.log = logging.getLogger('EchoClient')
        self.f = future
```

当客户端成功链接上服务端，立即进行通信。一系列的消息同时发送，不过在网络代码之下可能组合多个消息一次发送。当所有的消息发送完，最后发送EOF。

虽然看起来数据是被立即发送出去的，事实上transport对象先缓存需要发出去的数据，并且建立回调，当socket缓存准备好发送数据时才实际发送。这些处理都是透明的，所以应用代码可以像刚才进行IO操作时一样。

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log.debug(
            'connecting to {} port {}'.format(*self.address)
        )
        # This could be transport.writelines() except that
        # would make it harder to show each part of the message
        # bing sent.
        for msg in self.messages:
            transport.write(msg)
            self.log.debug('sending {!r}'.format(msg))
        if transport.can_write_eof():
            transport.write_eof()
```

当接收到服务端的响应时，打印日志。

```python
    def data_received(self, data):
        self.log.debug('received {!r}'.format(data))
```

当接收到EOF标记或者服务端关闭了链接，本地的transport对象也被关闭，future对象通过设置结果被标记完成。

```python
    def eof_received(self):
        self.log.debug('received EOF')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)

    def connection_lost(self, exc):
        self.log.debug('server closed connection')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)
        super().connection_lost(exc)
```

通常protocol类被传递给事件循环来创建连接。在这个例子中，因为事件循环传递其他的参数给protocol构造器，所以需要创建一个partial来包裹client类和发送的消息列表和Future实例。当调用create_connection()建立客户端链接时，使用新的可调用的类来代替。

```python
client_completed = asyncio.Future()

client_factory = functools.partial(
    EchoClient,
    messages=MESSAGES,
    future=client_completed,
)
factory_coroutine = event_loop.create_connection(
    client_factory,
    *SERVER_ADDRESS,
)
```

为了触发客户端运行，使用协程创建客户端时调用事件循环一次，然后在客户端完成时使用Future实例完成通信。像这样使用两个调用是避免客户端程序在完成服务端连接退出时进入死循环。如果只使用第一个调用等待协程创建客户端，则不可能处理所有的返回数据和清理服务端的链接。

```python
log.debug('waiting for client to complete')
try:
    event_loop.run_until_complete(factory_coroutine)
    event_loop.run_until_complete(client_completed)
finally:
    log.debug('closing event loop')
    event_loop.close()
```

在一个窗口运行服务端，在另一个窗口运行客户端产生下面的输出。

```
$ python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop

$python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connectiong to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop

$ python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 prot 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop
```

尽管客户端一直独立的发送消息，但是客户端第一次运行接收到服务端返回的大量数据。这些响应数据在后续运行中有所不同, 这取决于网络拥挤和在所有数据准备好之后进行网络缓存刷新。

```
$ python3 asyncio_echo_server_protocol.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
EchoServer_::1_63347: connection accepted
EchoServer_::1_63347: received b'This is the message. It will be sent in parts.'
EchoServer_::1_63347: received EOF
EchoServer_::1_63347: closing

EchoServer_::1_63387: connection accepted
EchoServer_::1_63387: received b'This is the message. '
EchoServer_::1_63387: sent b'This is the message. '
EchoServer_::1_63387: received b'It will be sent in parts.'
EchoServer_::1_63387: sent b'It will be sent in parts.'
EchoServer_::1_63387: received EOF
EchoServer_::1_63387: closing

EchoServer_::1_63389: connection accepted
EchoServer_::1_63389: received b'This is the message. It will be sent '
EchoServer_::1_63389: sent b'This is the message. It will be sent '
EchoServer_::1_63389: received b'in parts.'
EchoServer_::1_63389: sent b'in parts.'
EchoServer_::1_63389: received EOF
EchoServer_::1_63389: closing
```

[原文链接](https://pymotw.com/3/asyncio/io_protocol.html#output)
