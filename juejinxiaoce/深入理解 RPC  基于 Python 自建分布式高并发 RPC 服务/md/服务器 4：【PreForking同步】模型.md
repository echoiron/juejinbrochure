# 服务器 4：【PreForking同步】模型

进程要比线程更加吃资源，如果来一个连接就开一个进程，当连接比较多时，进程数量也会跟着多起来，操作系统的调度压力也就会比较大。所以我们要对服务器开辟的进程数量进行限制，避免系统负载过重。这就需要掌握多进程 PreForking 模型。

## 多进程 PreForking 模型

采用 PreForking 模型可以对子进程的数量进行了限制。PreForking 是通过预先产生多个子进程，共同对服务器套接字进行竞争性的 accept，当一个连接到来时，每个子进程都有机会拿到这个连接，但是最终只会有一个进程能 accept 成功返回拿到连接。子进程拿到连接后，进程内部可以继续使用单线程或者多线程同步的形式对连接进行处理。

![](https://user-gold-cdn.xitu.io/2018/5/16/163686ba498563be?w=722&h=383&f=png&s=31336)

下面的例子中，我们的子进程内部使用单线程对连接进行处理。

```
# coding: utf8
# prefork.py

import os
import json
import struct
import socket


def handle_conn(conn, addr, handlers):
    print addr, "comes"
    while True:
        length_prefix = conn.recv(4)
        if not length_prefix:
            print addr, "bye"
            conn.close()
            break  # 关闭连接，继续处理下一个连接
        length, = struct.unpack("I", length_prefix)
        body = conn.recv(length)
        request = json.loads(body)
        in_ = request['in']
        params = request['params']
        print in_, params
        handler = handlers[in_]
        handler(conn, params)


def loop(sock, handlers):
    while True:
        conn, addr = sock.accept()
        handle_conn(conn, addr, handlers)


def ping(conn, params):
    send_result(conn, "pong", params)


def send_result(conn, out, result):
    response = json.dumps({"out": out, "result": result})
    length_prefix = struct.pack("I", len(response))
    conn.sendall(length_prefix)
    conn.sendall(response)


def prefork(n):
    for i in range(n):
        pid = os.fork()
        if pid < 0:  # fork error
            return
        if pid > 0:  # parent process
            continue
        if pid == 0:
            break  # child process


if __name__ == '__main__':
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(("localhost", 8080))
    sock.listen(1)
    prefork(10)  # 好戏在这里，开启了 10 个子进程
    handlers = {
        "ping": ping
    }
    loop(sock, handlers)

```

如果并行的连接数超过了 prefork 进程的数量，那么后来的客户端请求将会阻塞，因为正在处理连接的子进程是没有机会去调用 accept 来获取新连接的。为了不阻塞新的客户端，我们可以将子进程的单线程同步模型改成多线程同步模型即可。

![](https://user-gold-cdn.xitu.io/2018/5/16/163686cd2e845425?w=935&h=442&f=png&s=53008)

## accept 竞争

prefork 之后，父进程创建的服务套接字引用，每个子进程也会继承一份，它们共同指向了操作系统内核的套接字对象，共享了同一份连接监听队列。子进程和父进程一样都可以对服务套接字进行 accept 调用，从共享的监听队列中摘取一个新连接进行处理。

## 小结

到目前为止，你们已经掌握了很多自己从来未曾听过的技术，但是我要说明的是这些知识还只能称得上是古典的 RPC 并发技术。

![](https://user-gold-cdn.xitu.io/2018/5/19/16376b94eac96ce5?w=300&h=263&f=jpeg&s=9330)

下一节我们要开始使用现代化技术来武装自己——异步高并发。

## 练习

PreForking 是进程池模型，相对应的线程池模型，我们目前并没有提及。读者是不是可以自己尝试完成一个线程池模型的 RPC 服务器呢？