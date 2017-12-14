---
title: asyncio之使用SSL
date: 2017-10-23 09:50:35
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

asyncio内置支持在套接字上启用SSL通信。通过将SSLContext实例传递给协程来创建服务器或者客户端连接，可支持并确保在将套接字交给应用程序使用之前启用SSL协议。

比以前章节中以协程为基础的服务端和客户端更新了一些小变化。首先创建证书和key文件。使用下面命令生成自签名证书。

```sh
$ openssl req -newkey rsa:2048 -nodes -keyout pymotw.key -x509 -days 365 -out pymotw.crt
```

openssl命令将提示生成证书的几个值，然后生成需要的输出文件。

在以前的服务端例子中，不安全的socket使用start_server()来创建监听的套接字。

```python
factory = asyncio.start_server(echo, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
```

为了加入编码认证，需要创建带证书和带刚生成key的SSLContext，然后传递给start_server()方法。

```python
# The certificate is created with pymotw.com as the hostname,
# which will not match when the example code runs elsewhere,
# so disable hostname verification
ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ssl_context.check_hostname = False
ssl_context.load_cert_chain('pymotw.crt', 'pymotw.key')

# Create the server and let the loop finish the coroutine before
# starting the real event loop.
factory = asyncio.start_server(echo, *SERVER_ADDRESS, ssl=ssl_context)
```
客户端需要相似的改变。老版本使用open_connection()来创建套接字链接到服务端。

```python
    reader, writer = await asyncio.open_connection(*address)
```

也需要SSLContext来保护客户端的socket。客户端身份不需要强制，只需要加载证书。

```python
    # The certificate is created with pymotw.com as the hostname,
    # which will not match when the example code runs
    # elsewhere, so disable hostname verification.
    ssl_context = ssl.create_default_context(
        ssl.Purpose.SERVER_AUTH,
    )
    ssl_context.check_hostname = False
    ssl_context.load_verify_locations('pymotw.crt')
    reader, writer = await asyncio.open_connection(
        *server_address, ssl=ssl_context
    )
```

在客户端需要一个其他小的改变。因为SSL连接不支持发送end-of-file(EOF)，客户端使用NULL字节作为消息终止。

老版本的客户端使用write_eof()发送给loop。

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

新版本发送零字节(b'\x00').

```python
    # This could be writer.writelines() except that
    # would make it harder to show each part of the message
    # being sent.
    for msg in messages:
        writer.write(msg)
        log.debug('sending {!r}'.format(msg))
    # SSL does not support EOF, so send a null byte to indicate
    # the end of the message.
    writer.write(b'\x00')
    await writer.drain()
```

服务端的echo()协程方法必须寻找NULL字节，在接收到时关闭客户端连接。

```python
async def echo(reader, writer):
    address = writer.get_extra_info('peername')
    log = logging.getLogger('echo_{}_{}'.format(*address))
    log.debug('connection accepted')
    while True:
        data = await reader.read(128)
        terminate = data.endswith(b'\x00')
        data = data.rstrip(b'\x00')
        if data:
            log.debug('received {!r}'.format(data))
            writer.write(data)
            await writer.drain()
            log.debug('sent {!r}'.format(data))
        if not data or terminate:
            log.debug('message terminated, closing connection')
            writer.close()
            return
```

在一个窗口启动服务端，在另一个窗口运行客户端，如下输出。

```
$ python3 asyncio_echo_server_ssl.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
echo_::1_53957: connection accepted
echo_::1_53957: received b'This is the message. '
echo_::1_53957: sent b'This is the message. '
echo_::1_53957: received b'It will be sent in parts. '
echo_::1_53957: sent b'It will be sent in parts.'
echo_::1_53957: message terminated, closing connection

$ python3 asyncio_echo_client_ssl.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message.'
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message.'
echo_client: received b'It will be sent in parts.'
echo_client: closing
main: closing event loop
```

[原文链接](https://pymotw.com/3/asyncio/ssl.html)
