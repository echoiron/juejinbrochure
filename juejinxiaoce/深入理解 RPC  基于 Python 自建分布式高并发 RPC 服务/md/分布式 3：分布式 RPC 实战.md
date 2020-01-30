# 分布式 3：分布式 RPC 实战

分布式 RPC 服务开发实战这个大作业将分为服务器和客户端两个模块，每个模块都不是特别简单。

服务器要进行复杂的进程管理，客户端要进行复杂的服务列表变更处理。最终我们将打造出一个可用的高性能分布式 RPC 服务，服务器可以横向扩展到多个机器多个节点，客户端可以实时随着服务器的变动而动态切换调用地址。

下面这张图就是本节大作业的全貌。

![](https://user-gold-cdn.xitu.io/2018/5/31/163b40840eca31e5?w=1134&h=475&f=png&s=95754)

我们先完成左半部分：

*   实现出一个 PreForking 异步模型的单机 RPC 服务器；
*   然后将服务挂接到 ZooKeeper 的树节点上；
*   再编写客户端消费者从 ZooKeeper 中读取服务节点地址，连接 RPC 服务器进行交互；
*   同时还要监听 ZooKeeper 树节点的变更，在 RPC 服务器节点变动时能动态调整服务列表地址。
*   单机服务器的内容会比之前要复杂一些，因为要考虑周全，对子进程进行管理，要处理信号监听和子进程收割等，这也是对上节理论内容的实战开发应用。

本节我们还将增加一个新的 RPC 服务，计算圆周率，使得我们的 RPC 服务不再过于单调。首先我们看圆周率公式。

![](https://user-gold-cdn.xitu.io/2018/5/31/163b3f5b64ff9974?w=1288&h=194&f=png&s=31781)

我们通过增大 n 的值，级数的值就越来越接近圆周率，n 的值将作为圆周率服务的参数。

## 完整的 RPC 服务器

继上节的异步 prefork 服务器，下面的代码增加了服务发现、子进程收割、信号处理功能。异步多进程服务共享同样的监听地址，所以只需要父进程注册服务即可。

*   父进程需要设置 SIGCHLD 信号处理函数收割意外退出的子进程，避免僵尸进程。
*   父进程需要在进程退出之前杀死所有子进程并收割之。
*   父进程需要在退出时关闭 zk 会话，立即释放临时节点。
*   父进程需要考虑 waitpid 被其它信号处理函数打断时进行重试。
*   父进程在杀死子进程时有可能遇到子进程已经提前死掉了，这时会爆出异常需要进行捕获。

```
# coding: utf8

import os
import sys
import math
import json
import errno
import struct
import signal
import socket
import asyncore
from cStringIO import StringIO
from kazoo.client import KazooClient


class RPCHandler(asyncore.dispatcher_with_send):

    def __init__(self, sock, addr):
        asyncore.dispatcher_with_send.__init__(self, sock=sock)
        self.addr = addr
        self.handlers = {
            "ping": self.ping,
            "pi": self.pi
        }
        self.rbuf = StringIO()

    def handle_connect(self):
        print self.addr, 'comes'

    def handle_close(self):
        print self.addr, 'bye'
        self.close()

    def handle_read(self):
        while True:
            content = self.recv(1024)
            if content:
                self.rbuf.write(content)
            if len(content) < 1024:
                break
        self.handle_rpc()

    def handle_rpc(self):
        while True:
            self.rbuf.seek(0)
            length_prefix = self.rbuf.read(4)
            if len(length_prefix) < 4:
                break
            length, = struct.unpack("I", length_prefix)
            body = self.rbuf.read(length)
            if len(body) < length:
                break
            request = json.loads(body)
            in_ = request['in']
            params = request['params']
            print os.getpid(), in_, params
            handler = self.handlers[in_]
            handler(params)
            left = self.rbuf.getvalue()[length + 4:]
            self.rbuf = StringIO()
            self.rbuf.write(left)
        self.rbuf.seek(0, 2)

    def ping(self, params):
        self.send_result("pong", params)

    def pi(self, n):
        s = 0.0
        for i in range(n+1):
            s += 1.0/(2*i+1)/(2*i+1)
        result = math.sqrt(8*s)
        self.send_result("pi_r", result)

    def send_result(self, out, result):
        response = {"out": out, "result": result}
        body = json.dumps(response)
        length_prefix = struct.pack("I", len(body))
        self.send(length_prefix)
        self.send(body)


class RPCServer(asyncore.dispatcher):

    zk_root = "/demo"
    zk_rpc = zk_root + "/rpc"

    def __init__(self, host, port):
        asyncore.dispatcher.__init__(self)
        self.host = host
        self.port = port
        self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
        self.set_reuse_addr()
        self.bind((host, port))
        self.listen(1)
        self.child_pids = []
        if self.prefork(10):  # 产生子进程
            self.register_zk()  # 注册服务
            self.register_parent_signal()  # 父进程善后处理
        else:
            self.register_child_signal()  # 子进程善后处理

    def prefork(self, n):
        for i in range(n):
            pid = os.fork()
            if pid < 0:  # fork error
                raise
            if pid > 0:  # parent process
                self.child_pids.append(pid)
                continue
            if pid == 0:
                return False  # child process
        return True

    def register_zk(self):
        self.zk = KazooClient(hosts='127.0.0.1:2181')
        self.zk.start()
        self.zk.ensure_path(self.zk_root)  # 创建根节点
        value = json.dumps({"host": self.host, "port": self.port})
        # 创建服务子节点
        self.zk.create(self.zk_rpc, value, ephemeral=True, sequence=True)

    def exit_parent(self, sig, frame):
        self.zk.stop()  # 关闭 zk 客户端
        self.close()  # 关闭 serversocket
        asyncore.close_all()  # 关闭所有 clientsocket
        pids = []
        for pid in self.child_pids:
            print 'before kill'
            try:
                os.kill(pid, signal.SIGINT)  # 关闭子进程
                pids.append(pid)
            except OSError, ex:
                if ex.args[0] == errno.ECHILD:  # 目标子进程已经提前挂了
                    continue
                raise ex
            print 'after kill', pid
        for pid in pids:
            while True:
                try:
                    os.waitpid(pid, 0)  # 收割目标子进程
                    break
                except OSError, ex:
                    if ex.args[0] == errno.ECHILD:  # 子进程已经割过了
                        break
                    if ex.args[0] != errno.EINTR:
                        raise ex  # 被其它信号打断了，要重试
            print 'wait over', pid

    def reap_child(self, sig, frame):
        print 'before reap'
        while True:
            try:
                info = os.waitpid(-1, os.WNOHANG)  # 收割任意子进程
                break
            except OSError, ex:
                if ex.args[0] == errno.ECHILD:
                    return  # 没有子进程可以收割
                if ex.args[0] != errno.EINTR:
                    raise ex  # 被其它信号打断要重试
        pid = info[0]
        try:
            self.child_pids.remove(pid)
        except ValueError:
            pass
        print 'after reap', pid

    def register_parent_signal(self):
        signal.signal(signal.SIGINT, self.exit_parent)
        signal.signal(signal.SIGTERM, self.exit_parent)
        signal.signal(signal.SIGCHLD, self.reap_child)  # 监听子进程退出

    def exit_child(self, sig, frame):
        self.close()  # 关闭 serversocket
        asyncore.close_all()  # 关闭所有 clientsocket
        print 'all closed'

    def register_child_signal(self):
        signal.signal(signal.SIGINT, self.exit_child)
        signal.signal(signal.SIGTERM, self.exit_child)

    def handle_accept(self):
        pair = self.accept()  # 接收新连接
        if pair is not None:
            sock, addr = pair
            RPCHandler(sock, addr)


if __name__ == '__main__':
    host = sys.argv[1]
    port = int(sys.argv[2])
    RPCServer(host, port)
    asyncore.loop()  # 启动事件循环

```

## 完整的 RPC 客户端

我们给上节的客户端代码增加了获取服务列表功能，并持续监听服务列表变更，然后循环随机选出一个可用服务器发送 ping 指令和 pi 指令输出服务器反馈。

当服务列表变更时，我们需要将新的服务列表和内存中现有的服务列表进行比对，创建新的连接，关闭旧的连接。

```
# coding: utf-8

import json
import time
import struct
import socket
import random
from kazoo.client import KazooClient

zk_root = "/demo"

G = {"servers": None}  # 全局变量，RemoteServer 对象列表


class RemoteServer(object):  # 封装 rpc 套接字对象

    def __init__(self, addr):
        self.addr = addr
        self._socket = None

    @property
    def socket(self):  # 懒惰连接
        if not self._socket:
            self.connect()
        return self._socket

    def ping(self, twitter):
        return self.rpc("ping", twitter)

    def pi(self, n):
        return self.rpc("pi", n)

    def rpc(self, in_, params):
        sock = self.socket
        request = json.dumps({"in": in_, "params": params})
        length_prefix = struct.pack("I", len(request))
        sock.send(length_prefix)
        sock.sendall(request)
        length_prefix = sock.recv(4)
        length, = struct.unpack("I", length_prefix)
        body = sock.recv(length)
        response = json.loads(body)
        return response["out"], response["result"]

    def connect(self):
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        host, port = self.addr.split(":")
        sock.connect((host, int(port)))
        self._socket = sock

    def reconnect(self):  # 重连
        self.close()
        self.connect()

    def close(self):
        if self._socket:
            self._socket.close()
            self._socket = None


def get_servers():
    zk = KazooClient(hosts="127.0.0.1:2181")
    zk.start()
    current_addrs = set()  # 当前活跃地址列表

    def watch_servers(*args):  # 闭包函数
        new_addrs = set()
        # 获取新的服务地址列表，并持续监听服务列表变动
        for child in zk.get_children(zk_root, watch=watch_servers):
            node = zk.get(zk_root + "/" + child)
            addr = json.loads(node[0])
            new_addrs.add("%s:%d" % (addr["host"], addr["port"]))
        # 新增的地址
        add_addrs = new_addrs - current_addrs
        # 删除的地址
        del_addrs = current_addrs - new_addrs
        del_servers = []
        # 先找出所有的待删除 server 对象
        for addr in del_addrs:
            for s in G["servers"]:
                if s.addr == addr:
                    del_servers.append(s)
                    break
        # 依次删除每个 server
        for server in del_servers:
            G["servers"].remove(server)
            current_addrs.remove(server.addr)
        # 新增 server
        for addr in add_addrs:
            G["servers"].append(RemoteServer(addr))
            current_addrs.add(addr)

    # 首次获取节点列表并持续监听服务列表变更
    for child in zk.get_children(zk_root, watch=watch_servers):
        node = zk.get(zk_root + "/" + child)
        addr = json.loads(node[0])
        current_addrs.add("%s:%d" % (addr["host"], addr["port"]))
    G["servers"] = [RemoteServer(s) for s in current_addrs]
    return G["servers"]


def random_server():  # 随机获取一个服务节点
    if G["servers"] is None:
        get_servers()  # 首次初始化服务列表
    if not G["servers"]:
        return
    return random.choice(G["servers"])


if __name__ == '__main__':
    for i in range(100):
        server = random_server()
        if not server:
            break  # 如果没有节点存活，就退出
        time.sleep(0.5)
        try:
            out, result = server.ping("ireader %d" % i)
            print server.addr, out, result
        except Exception, ex:
            server.close()  # 遇到错误，关闭连接
            print ex
        server = random_server()
        if not server:
            break  # 如果没有节点存活，就退出
        time.sleep(0.5)
        try:
            out, result = server.pi(i)
            print server.addr, out, result
        except Exception, ex:
            server.close()  # 遇到错误，关闭连接
            print ex

```

## 运行

```
# 开一个服务端窗口
python server.py localhost 8080
# 再开一个服务端窗口
python server.py localhost 8081
# 开一个客户端窗口
python client.py
# 再开一个客户端窗口
python client.py

```

运行结果如下

```
...
localhost:8888 pong ireader 93
localhost:8888 pi_r 3.1382045832
localhost:8888 pong ireader 94
localhost:8888 pi_r 3.13824026548
localhost:8888 pong ireader 95
localhost:8888 pi_r 3.13827520401
localhost:8888 pong ireader 96
localhost:8888 pi_r 3.13830942181
localhost:8888 pong ireader 97
localhost:8888 pi_r 3.13834294093
localhost:8888 pong ireader 98
localhost:8888 pi_r 3.13837578257
localhost:8888 pong ireader 99
localhost:8888 pi_r 3.13840796707

```

我们看到两个服务地址都正确返回了响应，随着参数 n 的逐渐增大，pi 的值越来越接近圆周率。

另外如果任意关闭一个服务节点，可以看到客户端仍然可以持续正常输出，观察输出的服务地址只剩一个了。如果再重新启动这个服务节点，客户端的输出地址自动恢复到正常的两个地址。

## 小结

本节内容是 RPC 实战开发的最后一节，也是最难的一节。它涉及的知识点非常繁多。请读者务必保持耐心，对不熟悉的技术知识进行各个击破。理解本节代码需要读者具备一定的操作系统基础知识，还有 ZooKeeper 服务发现的使用方法。

如果你完全理解了以上代码，说明你的水平已经进入了高级层次，这些知识在你身边至少有 90% 以上的 Python 程序员都不具备，你已经正式踏上了少有人走的路。

后面我们将对最有代表性的两个开源 RPC 框架进行逐个讲解。从工程角度，使用这些开源的、成熟的大型互联网公司造出来的轮子是一个非常经济的选择。而且你已经通过「造轮子」深入理解了 RPC 原理，再应用这些开源框架，也会感到更加得心应手。

## 练习

请读者将 zk 换成 etcd 来完成 RPC 服务的分布式化，这要求读者学会搭建 etcd 服务器。简单化意见，读者可以使用 Docker 来启动 etcd 服务器。