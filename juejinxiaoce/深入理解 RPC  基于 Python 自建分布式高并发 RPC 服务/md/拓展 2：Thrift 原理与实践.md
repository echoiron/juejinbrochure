# 拓展 2：Thrift 原理与实践

国外开源影响力大厂除了 Google 就数 Facebook 了。自从 Google 开源了 protobuf 之后，Facebook 紧随其后，开源了自家的 Thrift RPC 框架。相比 protobuf 一个单纯的序列化工具而言，thrift 是一个全套的 RPC 框架，支持多种协议，多种传输模式和多种服务器模型，还有单向消息，多种容器型数据结构和自定义异常类型。

![](https://user-gold-cdn.xitu.io/2018/6/5/163cecbb9ea989dc?w=400&h=400&f=png&s=59433)

上面这张图是官方的 Thrift 结构图 (老师还是觉得自己画的图好看)。从下往上看，TTransport 层可以选择多种传输模式，TProtocol 层可以选择多种协议，再上面两层是 Thrift 编译器生成的代码，包含消息的编码解码器、客户端服务器的 Stub 类和服务器的实现接口。最上层服务器这边要自己实现服务器接口，组装服务器代码，客户端要包装 Stub 类来调用 RPC 远程服务。

## 协议选择

Thrift 支持多种协议，有文本协议有二进制协议。Python 版本的文本协议默认提供了 JSON 序列化的实现 TJSONProtocol，二进制协议有普通的二进制协议 TBinaryProtocol 和紧凑版的二进制协议 TCompactProtocol。用户可以自己扩展协议类型。下面的例子我们使用 TCompactProtocol，这也是官方推荐的协议，它在兼具性能的前提下还能有较高的空间效率。

## 服务器模式选择

Thrift 支持多种服务器模式，我们在实战环节列举的服务器模式它全部支持

1.  单线程模式
2.  多线程模式
3.  多进程模式
4.  线程池模式
5.  进程池模式
6.  异步模式

单看这一点，Thrift 要比 gRPC 更加有优势，gRPC 都不能很好地支持多进程模式。Thrift 默认没有提供多进程异步模式，这个可以由用户自己去扩展。

## 传输层选择

Thrift 支持多种传输模式，除了普通的 TCP socket 之外，还有对 ssl 的支持，有对压缩的支持。这些都可以由用户自己来选择。我们通常在传输层使用缓冲模式，在序列化消息时，待完整的消息结构体序列化完成后调用 flush 方法才会将消息传递到对方，有助于提升 IO 效率，避免消息的半包问题频繁打扰对方。

## Thrift 入门

![](https://user-gold-cdn.xitu.io/2018/6/5/163cefafce5878f8?w=899&h=489&f=png&s=78087)

Thrift 使用上和 gRPC 差不多，使用步骤图跟 gRPC 也没有区别，只不过是换了代码生成工具、依赖库和类命名。 下面我们使用 Thrift 来实现一边圆周率服务。

**编写协议文件**

```
namespace py pi  # 生成的 python 代码以 pi 作为包名

service PiService {
	double calc(1:i32 n)
}

```

Thrift 的协议文件要比 gRPC 简洁多了，参数和返回支持很多原生类型，gRPC 不行，都必须使用包装类型，所以对于输入和输出都要定义 message 结构体，比较繁琐。

为了和 gRPC 保持一致，我们将参数稍微复杂化一点，增加输入输出结构体，同时 Thrift 还支持自定义异常类型，当输入参数 n 非正数时，我们抛出 IllegalArgument 异常

```
namespace py pi

struct PiRequest {
1:i32 n
}

struct PiResponse {
1:double value
}

exception IllegalArgument {
1: string message
}

service PiService {
	PiResponse calc(1:PiRequest req) throws(1:IllegalArgument ia)
}

```

**生成代码**

```
thrift --gen py pi.thrift

```

执行后，我们发现目录下多了个 gen-py 目录，里面就是生成的代码。我们看目录里面有个 pi/PiService.py 文件，这个文件就包含了客户端服务器的 Stub，消息编码解码器和服务器接口。还有个 pi/PiService-remote 文件这个是用来测试 RPC 服务功能的客户端工具类，我们不用理会这个文件。然后还有 constants.py 和 ttypes.py, 一个是协议里定义的常量，一个是自定义结构体的编码解码类和自定义枚举类。Thrift 支持定义常量和枚举。

**服务器实现**

服务器我们使用线程池模式提供服务，当参数非法时，我们抛出自定义异常，期望客户端可以捕获。

```
# coding: utf-8
# server.py
import sys
sys.path.append('gen-py')  # 增加 py 包查找路径，这样才可以找到 pi 包
import math

from thrift.transport import TSocket, TTransport
from thrift.protocol import TCompactProtocol
from thrift.server import TServer

from pi.ttypes import PiResponse, IllegalArgument
from pi.PiService import Iface, Processor


class PiHandler(Iface):  # Iface 为 RPC 服务接口

    def calc(self, req):
        if req.n <= 0:
            raise IllegalArgument(message="parameter must be positive")
        s = 0.0
        for i in range(req.n):
            s += 1.0/(2*i+1)/(2*i+1)
        return PiResponse(value=math.sqrt(8*s))


if __name__ == '__main__':
    handler = PiHandler()
    processor = Processor(handler)
    transport = TSocket.TServerSocket(host="127.0.0.1", port=8888)
    tfactory = TTransport.TBufferedTransportFactory()  # 缓冲模式
    pfactory = TCompactProtocol.TCompactProtocolFactory()  # 紧凑模式

    # 线程池服务 RPC 请求
    server = TServer.TThreadPoolServer(processor, transport, tfactory, pfactory)
    # 设置线程池数量
    server.setNumThreads(10)
    # 设置线程为 daemon，当进程只剩下 daemon 线程时会立即退出
    server.daemon = True
    # 启动服务
    server.serve()

```

**客户端实现**

编写客户端时必须注意的是 Thrift 提供的客户端 Stub 不是线程安全的，连接通道并不具备像 gRPC 那样的 duplexing 多路复用的高级功能。当我们使用一个连接时，必须加锁，避免其它线程同时使用该连接。为了提升客户端性能，有必要包装一个客户端连接池来减少线程之间的竞争。

```
# coding: utf-8
# client.py
import sys
sys.path.append("gen-py")

from thrift.transport import TSocket, TTransport
from thrift.protocol import TCompactProtocol

from pi.PiService import Client
from pi.ttypes import PiRequest, IllegalArgument

if __name__ == '__main__':
    sock = TSocket.TSocket("127.0.0.1", 8888)
    transport = TTransport.TBufferedTransport(sock)  # 缓冲模式
    protocol = TCompactProtocol.TCompactProtocol(transport)  # 紧凑模式
    client = Client(protocol)

    transport.open()  # 开启连接
    for i in range(10):
        try:
            res = client.calc(PiRequest(n=i))
            print "pi(%d) =" % i, res.value
        except IllegalArgument as ia:  # 捕获异常
            print "pi(%d)" % i, ia.message
    transport.close()  # 关闭连接

```

**超时机制**

Thrift 的超时机制是通过套接字的 timeout 属性来控制读写超时的，gRPC 则是通过定时器来控制的。Thrift 客户端一旦出现超时，往往意味着当前连接需要关闭并重新建立新连接，才能保证链路数据的完整性。但是 gRPC 可以继续使用现有的通道而不用理会单个请求的超时问题，如果超时的消息又收到了，gRPC 的客户端 Stub 会自动丢弃这个迟到的消息。

## gRPC 还是 Thrift？

Python 版本的 gRPC 不支持多进程模型，这个是 gRPC 的缺点，不过 java 和 golang 版本的 gRPC 就不在乎多进程模型了。同时 gRPC 还不支持单向消息，在协议的效率上 gRPC 基于 HTTP2.0 协议，这个肯定是无法抗衡纯粹的二进制协议的。但是 gRPC 有它自己的特色之处在于它的客户端是多路复用的线程安全的，可以拿过来直接使用，而且多路复用的客户端在效率上有明显优势。而 Thrift 的客户端还需要用户自己去封装一个连接池才能进入可用状态。gRPC 虽然使用了稍微浪费流量的 HTTP2.0 协议，但是考虑到 HTTP 协议的广泛性，未来支持 HTTP2.0 的代理服务器中间件、负载均衡中间件等会越来越多，gRPC 可以直接透明地在这些中间件之间进行转发而无需进行复杂的协议转换工作。Thrift 就不行了，完全的二进制协议固然高效，但是兼容性就差的太远。

另外在开源影响力上，Google 一直比 Facebook 强势很多，我们使用的 Google 的开源项目要比 Facebook 多的太多。gRPC 出来的时间要比 Thrift 要晚，Google 自然是知道 gRPC 要比 Thrift 相比有明显的优势所以才敢放出来开源的。

还有在文档上，个人觉得 gRPC 的文档要优秀一些，至少在直观上要赏心悦目很多。我们去看 Thrift 的官方文档，还是老旧的多年以前的灰暗风格的文档，文档中有好多连接点击的时候居然 404，可见文档已经很久处于无人维护的状态。

但是 Thrift 的源码要简单很多，它的 py 版本几乎全是纯粹的 Python 语言编写的，如果要研究源码的话，还是应该选择 Thrift。读者可以尝试去看看 gRPC 的源码，最终都需要深入到底层 HTTP2.0 的 c 语言实现，代码巨大无比，老师我是看的很头疼，所以我也不建议读者去花费过多精力去研究。如果要深入 gRPC 的原理，还不如去 Github 上看 gRPC 的丰富的文档来得直接。

## 作业

1.  尝试使用其它服务器模型如多进程模型、异步模型来替换上面的多线程模型；
2.  编写一个简单的 Thrift 客户端连接池；
3.  对 Thrift 编写的 RPC 服务增加分布式服务发现功能；