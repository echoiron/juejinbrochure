# 分布式 2：分布式 RPC 知识基础

本节要学习一下 ZooKeeper 的进程管理、信号处理和服务发现的 Python 客户端基本使用。有了这些基础知识之后，大作业的代码理解起来就没有那么吃力了。请读者务必掌握了本节的所有知识点之后再去进行大作业，否则可能会感觉像天书一般难以理解。

![](https://user-gold-cdn.xitu.io/2018/6/3/163c48dc374d52f3?w=1248&h=488&f=png&s=58876)

## 杀死子进程

![](https://user-gold-cdn.xitu.io/2018/5/25/163960c4dc9966c6?w=457&h=143&f=png&s=9578)

Python 提供了 os.kill 函数，它可以向指定进程发送信号。比如你要强制杀死某个进程，可以向它发送 SIGKILL 信号。SIGKILL 信号比较暴力，对方进程会立即 crash。除了 SIGKILL，你也可以通过 SIGTERM 和 SIGINT 信号温柔地通知对方退出，只要对方进程设置了信号处理函数，就可以在退出之前执行一些清理工作。进程无法为 SIGKILL 信号设置处理函数，所以 SIGKILL 无法捕获，只能暴力退出，用户无法控制。

```
os.kill(pid, signal.SIGKILL)
os.kill(pid, signal.SIGTERM)
os.kill(pid, signal.SIGINT)

```

## 信号处理函数

![](https://user-gold-cdn.xitu.io/2018/6/12/163f2c0def73e10c?w=680&h=245&f=png&s=26000)

我们可以捕获特定信号，覆盖默认信号处理行为。比如终端的一个 Python 程序挂在那里，你敲击了键盘的 ctrl+c 往往会导致进程退出并抛出异常。比如下面这个例子，你在终端执行 `Python sig.py`，该进程会悬挂在那里休眠

```
# sig.py
import time

while True:
    print "hello"
    time.sleep(1)  # 休眠 1s

```

然后你敲击 `ctrl+c`

```
hello
hello
hello
^CTraceback (most recent call last):
  File "sig.py", line 12, in <module>
    time.sleep(1)
KeyboardInterrupt

```

进程被打断，爆出了 KeyboardInterrupt 异常。进程立即退出了。

原因就是当你敲击 ctrl+c 时，默认进程会收到 SIGINT(interrupt) 信号，该信号默认的处理函数就是退出进程。如果你为 SIGINT 设置了自定义函数，就可以控制进程不受 ctrl+c 影响。

```
import time
import signal


def ignore(sig, frame):  # 啥也不干，就忽略信号
    pass

signal.signal(signal.SIGINT, ignore)

while True:
    print "hello"
    time.sleep(1)

```

你试试狂按 ctrl+c，进程依旧打转，只是这 hello 输出的要比平时快一点，似乎不再受到 sleep 的影响。

为什么呢？因为 sleep 函数总是要被 SIGINT 信号打断的，不管你有没有设置信号处理函数，只不过因为有 while True 循环在保护着。

```
hello
hello
hello
^Chello
^Chello
^Chello
^Chello
^Chello
^Chello
^Chello
^Chello

```

## 错误码

Python 的 errono 包定义了很多操作系统调用错误码。比如 errno.EINTR 表示调用被打断，代码遇到此种错误时往往需要进行重试。还有 errno.ECHILD 在 waitpid 收割子进程时，目标进程不存在，就会有这样的错误。

下面列出了 errno 最常见的一些异常说明，读者简单理解即可，无需深究。

```
errno.EPERM
Operation not permitted  # 操作不允许

errno.ENOENT
No such file or directory  # 文件没找到

errno.ESRCH
No such process  # 进程未找到

errno.EINTR
Interrupted system call  # 调用被打断

errno.EIO
I/O error  # I/O 错误

errno.ENXIO
No such device or address  # 设备未找到

errno.E2BIG
Arg list too long  # 调用参数太多

errno.ENOEXEC¶
Exec format error  # exec 调用二进制文件格式错误

errno.EBADF
Bad file number  # 文件描述符错误

errno.ECHILD
No child processes  # 子进程不存在

errno.EAGAIN
Try again  # I/O 操作被打断，告知 I/O 操作重试

```

## 特殊信号

上面提到 SIGINT 信号一般指代键盘的 ctrl+c 触发的 Keyboard Interrupt。还有其它常见信号读者需要了解一下。

1.  SIGTERM 当你使用不带参数的 kill 指令杀死进程时，进程会收到该信号。此信号默认行为也是退出进程，但是允许用户自定义信号处理函数。
2.  SIGKILL 当你使用 `kill -9` 杀死进程时，进程会收到信号。此信号的处理函数无法覆盖，进程收到此信号会立即暴力退出。
3.  SIGCHLD 子进程退出时，父进程会收到此信号。当子进程退出后，父进程必须通过 waitpid 来收割子进程，否则子进程将成为僵尸进程，直到父进程也退出了，其资源才会彻底释放。

![](https://user-gold-cdn.xitu.io/2018/5/27/163a2003c32d9ac6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 收割子进程

收割子进程使用`os.waitpid(pid, options)`系统调用，可以提供具体 pid 来收割指定子进程，也可以通过参数 `pid=-1` 来收割任意子进程。

options 如果是 0，就表示阻塞等待子进程结束才会返回，如果是 WNOHANG 就表示非阻塞，有,就返回指定进程的 pid，没有,就返回 0。

waitpid 有可能抛出异常，如果指定 pid 进程不存在或者没有子进程可以收割，就会抛出 OSError(errno.ECHILD)，如果 waitpid 被其它信号打断，就会抛出 OSError(errno.EINTR)，这个时候可以选择重试。

## 父进程退出

当父进程退出时，它应该先关闭所有的子进程。它可以将 fork 调用的返回值 pid 记录下来，然后在进程退出之前调用 os.kill 挨个杀死它们。如果不关闭子进程，子进程将会继续运行，这可能不是你希望的行为。

```
pids = []
pid = os.fork()
if pid > 0:
    pids.append(pid)  # 记录子进程 pid
else:
    run_child()  # 运行子进程
    sys.exit(0)
for pid in pids:
    os.kill(pid, signal.SIGTERM) # 向子进程发送信号
for pid in pids:
    os.waitpid(pid, 0)  # 收割子进程
sys.exit(0)  # 父进程退出

```

## 信号连续打断

![](https://user-gold-cdn.xitu.io/2018/5/25/16396138a5e0611c?w=345&h=331&f=png&s=19416)

当我们正在执行一个信号处理函数时，可能又收到另外一个信号，该信号会打断当前正在执行的信号处理函数。

如上图所示，先来了第一个 SIGINT 信号，开始执行 SIGINT 信号的信号处理函数，信号处理函数执行到一半，又来了一个 SIGCHLD，然后开始执行 SIGCHLD 信号的信号处理函数，信号处理函数又只执行到了一半，又来了一个 SIGTERM 信号，所以又开始执行 SIGTERM 信号的信号处理函数，待 SIGTERM 信号处理函数执行完毕后，再返回 SIGCHLD 的信号处理函数继续执行，待 SIGCHLD 信号处理函数执行完毕后再返回到 SIGINT 信号处理函数继续执行。

如果信号处理函数中 waitpid 正在执行，这个时候突然来了一个 SIGINT 信号，那么待 SIGINT 信号处理函数执行完毕后返回到原来的 waitpid 调用时，waitpid 将会爆出 OSError 异常，也就是 waitpid 调用被打断了。

我们通过检查异常对象里面的错误类型，来决定是否要继续重试。

如果异常类型是 errno.EINTR，就可以继续重试 waitpid，如果是 errno.ECHILD，说明目标子进程已经结束了，遇到其它类型应该向上抛出异常。至于什么是其它异常，这个是没有具体定义的，而且是不应该出现的。

```
while True:
    try:
        os.waitpid(pid, 0)
        break  # 调用成功退出循环
    except OSError, ex:
        if ex.args[0] == errno.ECHILD:
            break # 目标子进程已经结束
        if ex.args[0] == errno.EINTR:
            continue  # EINTR 应该重试
        raise ex  # 其它异常，继续向上抛出

```

## 服务发现

ZooKeeper 的节点信息以树状结构存储在内存中。比如下面展示了服务发现的三个节点结构信息。每个节点内部可以存储一个字节串，节点用于服务发现时存储服务器的地址信息。

![](https://user-gold-cdn.xitu.io/2018/5/25/163962955da73a31?w=513&h=374&f=png&s=28715)

**顺序节点**

ZooKeeper 支持顺序节点 (sequence)，它可以在节点名称后面自动追加自增 id，避免节点名称重复。在服务发现中会有多个子节点，使用顺序节点可以很方便地增加子节点。

**临时节点**

ZooKeeper 支持临时节点 (ephemeral)，在会话结束时，临时节点会自动释放。之所以会用到临时节点是因为 ZooKeeper 的会话支持连接断开重连。短时间的连接断开并不会立即删除内存会话，而是有个过期时间，时间一到，会话会自动过期。可以显式发送会话结束指令强制关闭会话，如果客户端进程突然 crash 掉，来不及发送会话关闭指令，ZooKeeper 将通过会话自动过期机制关闭会话。

## kazoo

kazoo 是 Python 连接 ZooKeeper 的客户端 library。它对 zk 的 api 做了很好的封装，可以让我们非常便捷地使用 zk。我们将通过 kazoo 实现服务发现功能。

生产者负责生成服务发现的子节点，子节点会存储服务地址信息：

```
# 服务发现生产者
import json
import kazoo

zk = kazoo.KazooClient(hosts="localhost:2181")
zk.start()  # 启动客户端，尝试连接
value = json.dumps({"host": "127.0.0.1", "port": 8080})
zk.ensure_path("/demo")  # 确保根节点存在，如果没有会自动创建
# 创建顺序临时节点，这就是服务列表中的一个子服务地址信息
zk.create("/demo/rpc", value, ephemeral=True, sequence=True)
# 关闭 zk 会话，关闭客户端，否则临时节点不会立即消失
zk.stop()

```

消费者通过获取子节点列表来拿到服务地址：

```
# 服务发现消费者
import json
import kazoo

zk = KazooClient(hosts="127.0.0.1:2181")
zk.start()
servers = set()
for child in zk.get_children(zk_root):  # 获取子节点名称
    node = zk.get(zk_root + "/" + child)  # 获取子节点 value
    addr = json.loads(node[0])
    servers.add("%s:%d" % (addr["host"], addr["port"])
servers = list(servers)  # 专成列表

```

**监听服务节点变更**

ZooKeeper 提供了 watch 功能，在节点变动时，客户端可以收到通知，进而重加载服务列表。

```
def callback(*args):
    new_children = zk.get_children(zk_root, watch=callback)  # 继续 watch

children = zk.get_children(zk_root, watch=callback)  # 增加 watch 参数

```

我们在 callback 方法里重新调用 get\_children 以达到重加载服务列表的目的，然后还得继续 watch，持续监听服务节点变更。

## 小结

本节的知识点比较零散，它涉及到 Unix 操作系统和分布式数据库 ZooKeeper 的一些理论知识和使用细节。读者在阅读过程中可能会遇到障碍，这是正常现象。大家可以先使用搜索引擎寻找相关的资料进行深入了解，然后可以进入交流群进行讨论。

读者在掌握了本节的所有基础知识之后，接下来就可以进入终极实战环节 —— 编写分布式的 RPC 服务和客户端。