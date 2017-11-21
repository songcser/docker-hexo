---
title: 域名服务交互
date: 2017-10-25 02:41:41
tags:
---

应用程序使用网络与服务器进行通信时可需要域名服务(DNS)，如在主机名和IP地址之间进行转换。asyncio在事件循环上有方便的方法在后台进行处理，以避免在查询时阻塞。

## 根据域名查找地址

使用getaddrinfo()协程方法将主机名和端口号转换成IP和IPv6地址。和在socket模块中的方法版本一样，返回值是一个包含五个信息的tuple列表。

1. 地址族
2. 地址类型
3. 协议
4. 服务器的规范名称
5. 套接字地址元组适用于打开原始指定的端口上的服务器连接

查询可以通过协议进行过滤，在这个例子中，只返回TCP的响应。

```python
# asyncio_getaddrinfo.py

import asyncio
import logging
import socket
import sys

TARGETS = [
    ('pymotw.com', 'https'),
    ('doughellmann.com', 'https'),
    ('python.org', 'https'),
]

async def main(loop, targets):
    for target in targets:
        info = await loop.getaddrinfo(
            *target,
            proto=socket.IPPROTO_TCP,
        )
        for host in info:
            print('{:20}: {}'.format(target[0], host[4][0]))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, TARGETS))
finally:
    event_loop.close()
```
这个例子将主机名和协议名转换成IP地址和端口号

```
$ python3 asyncio_getaddrinfo.py

pymotw.com              : 66.33.211.242
doughellmann.com        : 66.33.211.240
python.org              : 23.253.135.79
python.org              : 2001:4802:7901::e60a:1375:0:6
```

## 根据地址查找域名

getnameinfo()协程方法以相反的方向工作，将IP地址转换成主机名并且可能将端口号转换成协议名。

```python
# asyncio_getnameinfo.py

import asyncio
import logging
import socket
import sys

TARGETS = [
    ('66.33.211.242', 443),
    ('104.130.43.121', 443)
]

async def main(loop, targets):
    for target in targets:
        info = await loop.getnameinfo(target)
        print('{:15}: {} {}'.format(target[0], *info))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, TARGETS))
finally:
    event_loop.close()
```

这个例子显示从域名公司DreamHost的服务器返回了pymotw.com的IP地址。第二个IP地址检查是python.org, 没有返回主机名。

```
$ python3 asyncio_getnameinfo.py

66.33.211.242   : apache2-echo.catalina.dreamhost.com https
104.130.43.121  : 104.130.43.121 https
```

[原文地址](https://pymotw.com/3/asyncio/dns.html)
