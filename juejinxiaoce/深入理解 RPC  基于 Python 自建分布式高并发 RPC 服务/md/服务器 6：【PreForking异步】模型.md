# 服务器 6：【PreForking异步】模型

单个进程的 IO 并发能力有限，虽然使用了事件轮询 API 和异步读写功能，但是还是不够应对大型服务的高并发要求。特别是 Python 这种语言因为 GIL 的存在使得单个进程只能榨干一个 CPU 核心。我们需要一种扩展机制可以扩大服务器的整体并发处理能力，好好利用现代处理器的多核优势，这就需要使用多进程。

将 PreForking 机制和事件轮询异步读写结合起来，就能达成上面扩展的目标。PreForking 出来的每个子进程内部都是一个事件循环，一个进程可以榨干一个 CPU 核心，多个进程就可以榨干多个 CPU 核心。

![](https://user-gold-cdn.xitu.io/2018/5/11/1634e13697d3b055?w=1033&h=639&f=png&s=76598)

## 多进程 PreForking 异步模型

代码实现和前面的单进程异步模型差别不大，就是多了个 prefork 调用。prefork 在服务器套接字启用监听队列之后进行，这样每个子进程都可以使用服务器套接字来获取新连接进行处理。

```
# coding: utf8

import os
import json
import struct
import socket
import asyncore
from cStringIO import StringIO


class RPCHandler(asyncore.dispatcher_with_send):

    def __init__(self, sock, addr):
        asyncore.dispatcher_with_send.__init__(self, sock=sock)
        self.addr = addr
        self.handlers = {
            "ping": self.ping
        }
        self.rbuf = StringIO()  # 读缓冲

    def handle_connect(self):
        print self.addr, 'comes'

    def handle_close(self):
        print self.addr, 'bye'
        self.close()

    def handle_read(self):
        while True:
            content = self.recv(1024)
            if content:
                self.rbuf.write(content)  # 追加到读缓冲
            if len(content) < 1024:  # 说明内核缓冲区空了，等待下个事件循环再继续读吧
                break
        self.handle_rpc()  # 处理新读到的消息

    def handle_rpc(self):
        while True:
            self.rbuf.seek(0)
            length_prefix = self.rbuf.read(4)
            if len(length_prefix) < 4:  # 半包
                break
            length, = struct.unpack("I", length_prefix)
            body = self.rbuf.read(length)
            if len(body) < length:  # 还是半包
                break
            request = json.loads(body)
            in_ = request['in']
            params = request['params']
            print os.getpid(), in_, params
            handler = self.handlers[in_]
            handler(params) # 处理 RPC
            left = self.rbuf.getvalue()[length + 4:]  # 截断读缓冲
            self.rbuf = StringIO()
            self.rbuf.write(left)
        self.rbuf.seek(0, 2)  # 移动游标到缓冲区末尾，便于后续内容直接追加

    def ping(self, params):
        self.send_result("pong", params)

    def send_result(self, out, result):
        response = {"out": out, "result": result}
        body = json.dumps(response)
        length_prefix = struct.pack("I", len(body))
        self.send(length_prefix)
        self.send(body)


class RPCServer(asyncore.dispatcher):

    def __init__(self, host, port):
        asyncore.dispatcher.__init__(self)
        self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
        self.set_reuse_addr()
        self.bind((host, port))
        self.listen(1)
        self.prefork(10)  # 开辟 10 个子进程

    def prefork(self, n):
        for i in range(n):
            pid = os.fork()
            if pid < 0:  # fork error
                return
            if pid > 0:  # parent process
                continue
            if pid == 0:
                break  # child process

    def handle_accept(self):
        pair = self.accept()  # 获取一个连接
        if pair is not None:
            sock, addr = pair
            RPCHandler(sock, addr)  # 处理连接


if __name__ == '__main__':
    RPCServer("localhost", 8080)
    asyncore.loop()

```

开源框架 Tornado 和开源代理服务器 Nginx 正是采用了多进程 PreForking 异步模型达到了业界啧啧称奇的高并发的处理能力。

## 同步模型 vs 异步模型

同步和异步的差别就好比卡车和摩托车一样，如果遇到了交通堵塞，卡车只能继续等待堵塞缓解才可以继续前进。但是摩托车不一样，它可以切换到其它小路不停地往前开。

## 小结

写到这里时，老师感觉自己的知识就要快被你们吸干了。

![](https://user-gold-cdn.xitu.io/2018/5/19/16376cfc8de560c2?w=225&h=225&f=jpeg&s=5484)

我本来打算还留一点给自己续命，但是好人做到底，我决定把这条老命全部豁出去了。

下一节我们正式开讲分布式系统构建知识。