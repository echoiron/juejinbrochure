# 基础篇：深入理解 RPC 交互流程

本节我们开始讲解 RPC 的消息交互流程，目的是搞清楚一个简单的 RPC 方法调用背后究竟发生了怎样复杂曲折的故事，以看透 RPC 的本质。

![](https://user-gold-cdn.xitu.io/2018/6/5/163cf789da84cb53?w=869&h=133&f=png&s=9766)

上图是信息系统交互模型宏观示意图，RPC 的消息交互则会深入到底层。

RPC 是两个子系统之间进行的直接消息交互，它使用操作系统提供的套接字来作为消息的载体，以特定的消息格式来定义消息内容和边界。

RPC 的客户端通过文件描述符的读写 API (read & write) 来访问操作系统内核中的网络模块为当前套接字分配的发送 (send buffer) 和接收 (recv buffer) 缓存。

![](https://user-gold-cdn.xitu.io/2018/5/31/163b4dcf06e0a780?w=865&h=471&f=png&s=40786)

如上图所示，左边的客户端进程写 RPC 指令消息到内核的发送缓存中，内核将发送缓存中的数据传送到物理硬件 NIC，也就是网络接口芯片 (Network Interface Circuit)。NIC 负责将翻译出来的模拟信号通过网络硬件传递到服务器硬件的 NIC。服务器的 NIC 再将模拟信号转成字节数据存放到内核为套接字分配的接收缓存中，最终服务器进程从接收缓存中读取数据即为源客户端进程传递过来的 RPC 指令消息。

消息从用户进程流向物理硬件，又从物理硬件流向用户进程，中间还经过了一系列的路由网关节点。

上图呈现的只是 RPC 一次消息交互的上半场，下半场是一个逆向的过程，从服务器进程向客户端进程返回响应数据。完整的一次 RPC 过程如下图所示：

![](https://user-gold-cdn.xitu.io/2018/10/17/166802bd272e86a2?w=811&h=432&f=png&s=50907)

下面用 Python 代码来描述上述过程。

*   Server 端死循环监听本地 8080 端口，等待客户端的连接。
*   客户端启动时连接本地 8080 端口，紧接着发送词一个字符串 hello，然后等待服务器响应。
*   服务器接收到客户端连接后立即收取客户端发送过来的字符串，也就是 hello，打印出来。然后立即给对方回复一个字符串 world。
*   客户端接收到服务器发送过来的 world，马上打印出来。
*   关闭连接，结束。

```
# coding: utf-8
# server

import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind(("localhost", 8080))
sock.listen(1)  # 监听客户端连接
while True:
    conn, addr = sock.accept()  # 接收一个客户端连接
    print conn.recv(1024)  # 从接收缓冲读消息 recv buffer
    conn.sendall("world")  # 将响应发送到发送缓冲 send buffer
    conn.close() # 关闭连接

# coding: utf-8
# client

import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("localhost", 8080))  # 连接服务器
sock.sendall("hello")  # 将消息输出到发送缓冲 send buffer
print sock.recv(1024)  # 从接收缓冲 recv buffer 中读响应
sock.close() # 关闭套接字


```

如果从上面代码上观察，我们其实很难看出上图所示的复杂过程。浮现在多数人脑海中往往是下面的这幅简约模型图。相比之下它要简单很多，这也正是操作系统设计的魅力所在，让你时时刻刻都在使用它却感受不到它的存在。

![](https://user-gold-cdn.xitu.io/2018/5/31/163b4e36daa5cc79?w=533&h=187&f=png&s=14580)

## 小结

通过本节内容，读者们对 RPC 的交互流程应该有了大致了解，但是还并不知道 RPC 之间到底交互了什么。就好比你能看到远方有几个人在说话，但是不知道他们在说啥。

![](https://user-gold-cdn.xitu.io/2018/5/19/163760c75f77399c?w=150&h=150&f=jpeg&s=6950)

下一节我们将放大细节，仔细观察 RPC 客户端服务器之间窃窃私语了什么，它们究竟是在用什么外星语言交流。

## 练习题

一个很有趣的小测试实验 , 请读者编写代码实现以下情景:

> 客户端疯狂发送请求，但是服务器不读不处理，会发生什么？