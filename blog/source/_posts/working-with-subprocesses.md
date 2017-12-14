---
title: asyncio之子进程交互
date: 2017-10-26 07:58:09
tags: [asyncio, 异步, 协程]
categories:
  - asyncio
---

经常需要和其他的一些程序和进程一起工作，以便使用已经存在的代码而不用重写，或者访问Python中已经不可用的库或特性。和网络IO一样，asyncio包含两个抽象概念来启动其他程序然后进行交互。

## 对于Subprocesses使用Protocol Abstraction

这个例子使用协程来启动一个进程运行Unix命令df，寻找磁盘的剩余空间。使用subprocess_exec()启动进程并把它绑定到protocol类，这样就知道如何读取df的命令输出并分析。当subprocess的IO事件发生时protocol类的方法被自动调用。由于stdin和stderr参数都设置为Nore，这些通信管道没有连接到新的进程。

```python
# asyncio_subprocess_protocol.py

import asyncio
import functools

async def run_df(loop):
    print('in run_df')

    cmd_done = asyncio.Future(loop=loop)
    factory = functools.partial(DFProtocol, cmd_done)
    proc = loop.subprocess_exec(
        factory,
        'df', '-hl',
        stdin=None,
        stderr=None,
    )
    try:
        print('launching process')
        transport, protocol = await proc
        print('waiting for process to complete')
        await cmd_done
    finally:
        transport.close()

    return cmd_done.result()
```

DFProtocol类由SubprocessProtocol派生出，定义类API和其他进程通过管道进行通信。done参数期望是Future对象，这样调用者可以监听到进程的完成。

```python
class DFProtocol(asyncio.SubprocessProtocol):
    
    FD_NAMES = ['stdin', 'stdout', 'stderr']

    def __init__(self, done_future):
        self.done = done_future
        self.buffer = bytearray()
        super().__init__()
```

和socket连接一样，当输入管道有新的进程建立时，connection_made()方法被调用。transport参数是BaseSubprocessTransport子类的实例。如果进程配置成可以接收输入，可以通过进程读取数据输出，并且写入数据到进程的输入流。

```python
    def connection_made(self, transport):
        print('process started {}'.format(transport.get_pid()))
        self.transport = transport
```

当进程产生输出，pipe_data_received()方法被调用，获取存储数据的文件描述符，并且从管道中读取真实数据。protocol类将进程的标准输出管道的输出保存在缓冲区中供以后处理。

```python
    def pipe_data_received(self, fd, data):
        print('read {} bytes from {}'.format(len(data), self.FD_NAMES[fd]))
        if fd == 1:
            self.buffer.extend(data)
```

当进程中断，process_exited()被调用。transport对象通过调用get_returncode()方法可以获取进程的退出码。在这种情况下，如果没有错误报告，则在通过Future实例返回之前，可用的输出将被解码分析。如果有错误，则结果被假定为空。进程退出时告诉run_df()方法设置future的结果，所有该方法清理并且返回结果。

```python
    def process_exited(self):
        print('process exited')
        return_code = self.transport.get_returncode()
        print('return code {}'.format(return_code))
        if not return_code:
            cmd_output = bytes(self.buffer).decode()
            results = self._parse_results(cmd_output)
        else:
            results = []
        self.done.set_result((return_code, results))
```

将命令输出解析成一系列字典，将每个输出行的标题名映射到它的值，并返回结果列表。

```python
    def _parse_results(self, output):
        print('parsing results')
        # Output has one row of headers, all single words.
        # remaining rows are one per filesystem, with columns
        # matching the headers (assuming that none of the
        # mount points have whitespace in the names)
        if not output:
            return []
        lines = output.splitlines()
        headers = lines[0].split()
        devices = lines[1:]
        results = [
            dict(zip(headers, line.split()))
            for line in devices
        ]
        return results
```

run_df()协程使用run_until_complete()方法运行，检测每一个设备的剩余空间并打印。

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(
        run_df(event_loop)
    )
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('\nFree space:')
    for r in results:
        print('{Mounted:25}: {Avail}'.format(**r))
```

下面的输出显示执行步骤的顺序，和运行系统上三个设备的剩余空间。

```
$ python3 asyncio_subprocess_protocol.py

in run_df
launching process
process started 49675
waiting for process to complete
read 332 bytes from stdout
process exited
return code 0
parsing results

Free space:
/                           : 233Gi
/Volumes/hubertinternal     : 157Gi
/Volumes/hubert-tm          : 2.3Ti
```

## 使用协程和流调用子进程

使用协程直接运行进程，而不是通过Protocol子类访问，请调用create_subprocess_exec()并且指定要连接到的管道stdout, stderr和stdin。协程生成子进程的结果是一个Process实例，可用于管理子进程或与之通信。

```python
# asyncio_subprocess_coroutine.py

import asyncio
import asyncio.subprocess

async def run_df():
    print('in run_df')

    buffer = bytearray()

    create = asyncio.create_subprocess_exec(
        'df', '-hl',
        stdout=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print('process started {}'.format(proc.pid))
```

在这个例子中， 除了命令行参数外，df不需要任何输入，所以下一步就是读取所有的输出。Protocol没有控制同一时间读取多少数据。这个例子使用readline()，但是也可以直接调用read()方法读取数据而不是面向行级的。和protocol例子一样，命令的输出进行了缓存，稍后打印出来。

```python
    while True:
        line = await proc.stdout.readline()
        print('read {!r}'.format(line))
        if not line:
            print('no more output from command')
            break
        buffer.extend(line)
```

因为程序完成，没有更多输出时，readline()方法返回空字符串。为了保证正确清理进程，下一步是等待进程完全退出。

```python
    print('waiting for process to complete')
    await proc.wait()
```

此时可以检查退出状态，以确定是解析输出还是错误处理，因为它不产生输出。逻辑解析和前面的例子是一样的，但是是独立的函数（这里没有显示），因为没有protocol类来隐藏它。数据解析之后，结果和退出码返回给调用者。

```python
    return_code = proc.returncode
    print('return code {}'.format(return_code))
    if not return_code:
        cmd_output = bytes(buffer).decode()
        results = _parse_results(cmd_output)
    else:
        results = []

    return (return_code, results)
```

主程序和基于protocol的示例看起来相似，因为实现的不同部分都是独立在run_df()方法中。

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(
        run_df()
    )
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('\nFree space:')
    for r in results:
        print('{Mounted:25}: {Avail}'.format(**r))
```

由于df的输出每次读取一行，所以可以不断的显示程序的进度。否则，输出看起来和以前的例子相似。

```
$ python3 asyncio_subprocess_coroutine.py

in run_df
launching process
process started 49678
read b'Filesystem   Size    Used    Avail   Capacity    iusedifree %iused Mounted on\n'
read b'/dev/disk2s2 446Gi   213Gi   233Gi   48%     5595508261015132     48%     /\n'
read b'/dev/disk1   465Gi   307Gi   157Gi   67%     8051492241281172     66%     /Volumes/hubertinternal\n'
read b'/dev/disk3s2 3.6Ti   1.4Ti   2.3Ti   38%     181837749306480579   37%     /Volumes/hubert-tm\n'
read b''
no more output from command
waiting for process to complete
return code 0
parsing results

Free space:
/                           : 233Gi
/Volumes/hubertinternal     : 157Gi
/Volumes/hubert-tm          : 2.3Ti
```

## 向子进程发送数据

以前的例子都是使用一个通信管道从第二个进程读取数据。经常需要向一个命令的进程发送数据。这个例子定义一个协程执行Unix命令tr来转换输入流中的字符。在这个示例中，tr用于将小写字母转换为大写字母。

to_upper()协程方法采用事件循环和字符串作为参数。它产生第二个进程运行"tr [:lower:] [:upper:]"

```python
# asyncio_subprocess_coroutine_write.py

import asyncio
import asyncio.subprocess

async def to_upper(input):
    print('in to_upper')

    create = asyncio.create_subprocess_exec(
        'tr', '[:lower:]', '[:upper:]',
        stdout=asyncio.subprocess.PIPE,
        stdin=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print('pid {}'.format(proc.pid))
```

下面to_upper()方法使用Process的communicate()方法发送输入字符串到命令行并且异步的读取所有输出结果。和subprocess.Popen版本相同的方法，communicate()方法返回完整的字符串字节输出。如果一个命令产生的数据不适合存放到内存中，那么输入不能一次产生或者输出必须立即处理掉，可以直接使用Process的stdin，stdout和stderr处理而不是调用communicate()。

```python
    print('communicating with process')
    stdout, stderr = await proc.communicate(input.encode())
```

IO完成之后，等待进程完全退出确保进程清理干净。

```python
    print('waiting for process to complete')
    await proc.wait()
```

检查退出码，解码输出字符串，准备好协程的返回值。

```python
    return_code = proc.returncode
    print('return code {}'.format(return_code))
    if not return_code:
        results = bytes(stdout).decode()
    else:
        results = ''

    return (return_code, results)
```

主程序创建message字符串进行转换，并且使用事件循环运行to_upper()和打印结果。

```python
MESSAGE = """
This message will be converted to all caps.
"""

event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(
        to_upper(MESSAGE)
    )
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('Original: {!r}'.format(MESSAGE))
    print('Changed : {!r}'.format(results))
```

输出显示了一系列操作和如何进行简单文字信息的转换。

```
$ python3 asyncio_subprocess_coroutine_write.py

in to_upper
launching process
pid 49684
communicating with process
waiting for process to complete
return code 0
Original: '\nThis message will be converted\nto all caps.\n'
Changed : '\nTHIS MESSAGE WILL BE CONVERTED\nTO ALL CAPS.\n'
```

[原文链接](https://pymotw.com/3/asyncio/subprocesses.html)
