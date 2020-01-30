# 拓展 1：gRPC 原理与实践

gRPC 是 Google 开源的基于 Protobuf 和 Http2.0 协议的通信框架，Google 深度学习框架 tensorflow 底层的 RPC 通信就完全依赖于 gRPC 库。Google 的开源影响力很大，目前已经有非常多的大型互联网公司在使用 gRPC 了。比如国产开源数据库影响力最大的 Tidb 就选择了 gRPC 作为底层通讯库。

![](https://user-gold-cdn.xitu.io/2018/6/4/163c8d8db8d4768e?w=654&h=226&f=png&s=15181)

gRPC 之所以选择 Http2.0 作为基础开源协议，是考虑到 Http 协议在互联网应用的广泛性。同时因为 Http2.0 支持的 Streaming 和 Duplexing 可以将请求和响应消息进行分片交叉传送，可以大幅提升传输效率，GRPC 特色的 Stream 消息正是使用了 Http2.0 的 Streaming 特性。

![](https://user-gold-cdn.xitu.io/2018/6/4/163c8d69cc6cc63b?w=668&h=196&f=png&s=9399)

为了理解 gRPC 的原理我们有必要先对 HTTP2.0 协议有个基本的了解。

## HTTP2.0

目前国内大部分互联网都还在使用 HTTP1.1 协议对外提供 API 服务，不过 HTTP2.0 在国外早已暗流涌动，很多大型互联网公司都已经有大量的业务线在使用了 HTTP2.0 协议。据 W3Techs 统计，截止 2018 年 6 月份，已经有 26% 的大型互联网服务已经开始使用 HTTP2.0 协议。HTTP2.0 协议比较复杂，下面我只讲一下 HTTP2.0 的一些重要特性。

**duplexing**

HTTP2.0 相比 HTTP1.1 有非常大的不同，HTTP1.1 还是基于文本协议的问答有序模式，但是 HTTP2.0 是基于二进制协议的乱序模式 (Duplexing)。这意味同一个连接通道上多个请求并行时，服务器处理快的可以先返回而不用因为等待其它请求的响应而排队。

![](https://user-gold-cdn.xitu.io/2018/6/4/163c8e7a99339e76?w=1009&h=340&f=png&s=47150)

**header 优化**

我们知道 HTTP 协议的请求头有大量的 key/value 文本组成，多个请求直接 key/value 重复程度很高。为了优化这部分，HTTP2.0 对请求头的 key/value 做了字典处理，对于常用的 key/value 文本无需重复传送，而是通过引用内部字典的整数索引来达到显著节省请求头传输流量的目的。这也意味着以后我们无法直接使用肉眼来观察 HTTP 协议测传输内容了，它的协议内容对肉眼不再十分友好。

**framing & streaming**

在 HTTP1.1 里面如果一个响应的内容太大，一般使用 chunk 模式进行分块传输。在 HTTP2.0 里面它也有分块传送，但是概念上叫着`Framing`——分帧传送。同一个响应会有同一个流 ID`stream_id`，消息接收端会将具有相同`stream_id`的消息帧串起来作为一个整体来处理，位于流的最后一个消息帧会有一个流结束标志位`(EOS——End of Stream)`。注意 EOS 并不意味着连接的中止，同一个连接上会有多个流穿插传输，结束了单个流还会有很多其它的流在继续传送。

![](https://user-gold-cdn.xitu.io/2018/6/4/163c8faa521c5315?w=986&h=84&f=png&s=18261)

**帧格式**

![](https://user-gold-cdn.xitu.io/2018/6/4/163c900e3019b85b?w=704&h=301&f=png&s=17078)

HTTP2.0 的消息都是分帧传送的，所有的消息都是一帧一帧的，有些小请求 / 响应只有一个帧，还有些大请求 / 响应因为内容多需要多个帧。每个帧的格式如上图所示，包含长度前缀、消息类型、消息标志位和消息内容 5 个部分。

HTTP2.0 默认定义了十种类型的帧，因为 type 字段是一个字节，可以支持 255 个类型，所以帧类型是可以扩展的。

1.  HEADERS 帧 头信息，对应于`HTTP HEADER`
2.  DATA 帧 对应于`HTTP Response Body`
3.  PRIORITY 帧 用于调整流的优先级
4.  RST\_STREAM 帧 流终止帧，用于中断资源的传输
5.  SETTINGS 帧 用户客户服务器交流连接配置信息
6.  PUSH\_PROMISE 帧 服务器向客户端主动推送资源
7.  GOAWAY 帧 礼貌地通知对方断开连接
8.  PING 帧 心跳帧，检测往返时间和连接可用性
9.  WINDOW\_UPDATE 帧 调整帧窗口大小
10.  CONTINUATION 帧 HEADERS 帧太大时分块的续帧

HTTP2.0 定义了三种常用的标志位，它可以理解为帧的属性。

1.  END\_STREAM 流结束标志，表示当前帧是流的最后一个帧
2.  END\_HEADERS 头结束表示，表示当前帧是头信息的最后一个帧
3.  PADDED 填充标志，在数据 Payload 里填充无用信息，用于干扰信道监听

通常我们最常用的 HTTP 请求 / 响应的帧形式如下

![](https://user-gold-cdn.xitu.io/2018/6/4/163c90e5a630d04d?w=1199&h=308&f=png&s=32875)

## **HTTP2.0 vs HTTP1.1**

HTTP2.0 和 HTTP1.1 的区别就好比政府企业和互联网企业。在政府企业里往往讲究论资排辈，谁先来谁是老大，后辈们只有等前辈们退休了才能继续往上走。而互联网企业不一样，谁有能力谁先上，所以我们经常看到某某公司里的 90 后的领导带着一屁股 80 后的兄弟们组队打怪做产品。

**protobuf**

我们在前面的 Protobuf 章节提到 Protobuf 协议仅仅定义了单个消息的序列化格式，我们需要自定义头部的长度和消息的类名称来区分消息边界。Protobuf 将消息序列化后，内容放在 HTTP2.0 的 Data 帧的 Payload 字段里，消息的长度在 Data 帧的 Length 字段里，消息的类名称放在 Headers 帧的 Path 头信息中。我们需要通过头部字段的 Path 来确定具体消息的解码器来反序列化 Data 帧的 Payload 内容，而这些工作 gRPC 都帮我们干了。

## gRPC 入门

![](https://user-gold-cdn.xitu.io/2018/6/4/163c96b5be229172?w=984&h=532&f=png&s=76848)

接下来我们使用 GRPC 来实现一下圆周率计算服务。整个过程分为五步

1.  编写协议文件 `pi.proto`
2.  使用`grpc_tools`工具将 `pi.proto`编译成`pi_pb2.py`和`pi_pb2_grpc.py`两个文件
3.  使用`pi_pb2_grpc.py`文件中的服务器接口类，编写服务器具体逻辑实现
4.  使用`pi_pb2_grpc.py`文件中的客户端 Stub，编写客户端交互代码
5.  分别运行服务器和客户端，观察输出结果

首先我们安装一下工具依赖

```
pip install grpcio_tools  # tools 包含代码生成工具，会自动安装依赖的 grpcio 包

```

**编写协议文件**

```
syntax = "proto3";

package pi;

// pi service
service PiCalculator {
    // pi method
    rpc Calc(PiRequest) returns (PiResponse) {}
}

// pi input
message PiRequest {
    int32 n = 1;
}

// pi output
message PiResponse {
    double value = 1;
}

```

协议文件包含输入消息`PiRequest`、输出消息`PiResponse`和 rpc 服务调用的定义`PiCalculator`

**生成代码**

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. pi.proto

```

这个命令行有很多参数，其中`python_out`目录指定`pi_pb2.py`文件的输出路径，`grpc_python_out`指定`pi_pb2_grpc.py`文件的输出路径。`-I`参数指定协议文件的查找目录，我们都将它们设置为当前目录。

命令执行后，可以看到当前目录下多了两个文件，`pi_pb2.py`和`pi_pb2_grpc.py`。前者是消息序列化类，后者包含了服务器 Stub 类和客户端 Stub 类，以及待实现的服务 RPC 接口。

**实现服务接口**

上面我们生成了服务的 RPC 接口，它只是个空接口，具体计算逻辑要用户自己来来实现。下面我们实现一下圆周率计算服务的逻辑。然后将服务类组装成一个完成的服务器。

![](https://user-gold-cdn.xitu.io/2018/5/31/163b3f5b64ff9974?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
# coding: utf-8
# server.py
import math
import grpc
import time
from concurrent import futures

import pi_pb2
import pi_pb2_grpc


# 圆周率计算服务实现类
class PiCalculatorServicer(pi_pb2_grpc.PiCalculatorServicer):

    def Calc(self, request, ctx):
        # 计算圆周率的逻辑在这里
        s = 0.0
        for i in range(request.n):
            s += 1.0/(2*i+1)/(2*i+1)
        # 注意返回的是一个响应对象
        return pi_pb2.PiResponse(value=math.sqrt(8*s))


def main():
    # 多线程服务器
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # 实例化圆周率服务类
    servicer = PiCalculatorServicer()
    # 注册本地服务
    pi_pb2_grpc.add_PiCalculatorServicer_to_server(servicer, server)
    # 监听端口
    server.add_insecure_port('127.0.0.1:8080')
    # 开始接收请求进行服务
    server.start()
    # 使用 ctrl+c 可以退出服务
    try:
        time.sleep(1000)
    except KeyboardInterrupt:
        server.stop(0)


if __name__ == '__main__':
    main()

```

**客户端**

客户端就简单多了，直接使用生成代码的客户端 Stub 就可以连接服务器进行调用了。我们运行 1000 次 Pi 计算服务，观察 pi 的计算结果是不是越来越接近圆周率。

```
# coding: utf-8
# client.py

import grpc

import pi_pb2
import pi_pb2_grpc


def main():
    channel = grpc.insecure_channel('localhost:8080')
    # 使用 stub
    client = pi_pb2_grpc.PiCalculatorStub(channel)
    # 调用吧
    for i in range(1, 1000):
        print "pi(%d) =" % i, client.Calc(pi_pb2.PiRequest(n=i)).value


if __name__ == '__main__':
    main()

```

**运行**

```
# 开一个窗口
python server.py
# 再开一个窗口
python client.py

```

观察到输出如下

```
...
pi(990) = 3.14127111202
pi(991) = 3.1412714365
pi(992) = 3.14127176033
pi(993) = 3.1412720835
pi(994) = 3.14127240602
pi(995) = 3.14127272789
pi(996) = 3.14127304912
pi(997) = 3.1412733697
pi(998) = 3.14127368964
pi(999) = 3.14127400894

```

**线程模型**

gRPC 默认使用的是异步 IO 模型，底层有一个独立的事件循环。它的异步不是使用 Python 内置的 asyncio 来完成的，它使用的是开源异步事件框架 gevent。gevent 的优势在于可以让用户使用同步的代码编写异步的逻辑，而完全不知内部正在进行复杂的协程调度，这样可以明显降低框架代码的实现复杂度。grpc 服务器端接收到一个完整的消息包后就会传递到线程池去进行业务逻辑处理，线程池是由用户层代码来指定的，待线程池将任务执行完成后会将结果扔到 IO 模型的完成队列中进行后续的响应处理。

![](https://user-gold-cdn.xitu.io/2018/6/4/163ca165f5549ee2?w=870&h=386&f=png&s=47110)

接下来我们改造一下客户端代码，使用客户端的并行 RPC 调用来计算圆周率。

```
# multithread_client.py
import grpc

import pi_pb2
import pi_pb2_grpc

from concurrent import futures

def pi(client, k):
    return client.Calc(pi_pb2.PiRequest(n=k)).value

def main():
    channel = grpc.insecure_channel('localhost:8080')
    # client 对象是线程安全的
    client = pi_pb2_grpc.PiCalculatorStub(channel)
    # 客户端使用线程池执行
    pool = futures.ThreadPoolExecutor(max_workers=4)
    results = []
    for i in range(1, 1000):
        results.append((i, pool.submit(pi, client, i)))
    # 等待所有任务执行完毕
    pool.shutdown()
    for i, future in results:
        print i, future.result()


if __name__ == '__main__':
    main()

```

现在我们去掉 print 打印输出，对比改造前后客户端的执行效率发现，两者执行时间居然差不多

```
> time python client.py
python client.py  0.43s user 0.10s system 44% cpu 1.192 total

> time python multithread_client.py
python multithread_client.py  0.57s user 0.21s system 60% cpu 1.289 total

```

这是为什么呢？原因在于 Python 服务器的 GIL 限制了程序最多只能使用单个 CPU 核心，虽然是多线程服务器，但是实际上对于纯计算性操作来说，多线程的执行效率跟单线程没有区别。下面我们对服务器进行一下改造，在计算方法里面增加一个 sleep 操作，比如现在我们在每个 Calc 方法里睡眠 0.01s，然后再对比，结果就会比较明显了。

```
> time python client.py
python client.py  0.68s user 0.15s system 5% cpu 14.031 total

> time python multithread_client.py
python multithread_client.py  0.67s user 0.25s system 43% cpu 2.099 total

```

在生产环境中，RPC 服务往往伴随着很多 IO 操作，多线程在 IO 型服务中可以明显提升整体服务效率。

**PreForking 模型**

注意 gRPC 暂时不支持多进程模式，如果要支持多进程模型，gRPC 官方表示还需要做大量的改造，目前 gRPC 还不支持 fork，笔者做了多次试验发现运行在 PreForking 模型下，客户端会出现莫名其妙的卡死现象，不建议读者使用。

## Streaming

gRPC 的一个特色之处在于提供了 Streaming 模式，有了 Streaming 模式，客户端可以将一连串的请求连续发送到服务器，服务器也可以将一连串连续的响应回复给客户端。它类似于列表类型的消息，但是又不一样。列表类的消息要求我们一次性将整个列表消息打包传递给服务器，但是 Streaming 不一样，可以生成一个请求就可以立即发往服务器，接下来继续生成请求，继续发送，就好比流水线一样。

对比前面的同步调用，Streaming 可以理解为 gRPC 的异步调用。

![](https://user-gold-cdn.xitu.io/2018/6/4/163ca3eb9c586cc5?w=747&h=218&f=png&s=20825)

Streaming 也是人类之间正常的对话形式，两个人对话可以一问一答，也可以一次性说很多话让对方一直听。不过 Streaming 还是不太一样，它允许一个人同时说又同时听，双工对话，人类在这方面的能力那是很欠缺的。

接下来我们将上面的圆周率服务改造成 Streaming 模式进行交互。

**编写协议文件**

```
syntax = "proto3";

package pi;

// pi service
service PiCalculator {
	// pi method
	rpc Calc(stream PiRequest) returns (stream PiResponse) {}
}

// pi input
message PiRequest {
	int32 n = 1;
}

// pi output
message PiResponse {
	int32  n = 1;
	double value = 2;
}

```

这个协议差别很小，就是在输入输出类型上增加了 stream 关键字就可以将输入输出同时变成流式了。不过还得注意到输出消息增加了一个参数，这个参数 n 就是输入的参数 n。当协议变成流式之后，输入和输出的对应关系需要在协议层面进行维护。因为不是所有的请求都会有响应，服务器可以忽略掉某些请求不进行处理，也就是输入和输出不再是一对一的关系。

**生成代码**

这个和前面的没有差别，一摸一样，读者拷贝粘贴即可。

**实现服务接口**

```
# coding: utf-8
# server.py
import math
import grpc
import time
import random
from concurrent import futures

import pi_pb2
import pi_pb2_grpc


class PiCalculatorServicer(pi_pb2_grpc.PiCalculatorServicer):

    def Calc(self, request_iterator, ctx):
        # request 是一个迭代器参数，对应的是一个 stream 请求
        for request in request_iterator:
            # 50% 的概率会有响应
            if random.randint(0, 1) == 1:
                continue
            s = 0.0
            for i in range(request.n):
                s += 1.0/(2*i+1)/(2*i+1)
            # 响应是一个生成器，一个响应对应对应一个请求
            yield pi_pb2.PiResponse(n=i, value=math.sqrt(8*s))


def main():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    servicer = PiCalculatorServicer()
    pi_pb2_grpc.add_PiCalculatorServicer_to_server(servicer, server)
    server.add_insecure_port('localhost:8080')
    server.start()
    try:
        time.sleep(1000)
    except KeyboardInterrupt:
        server.stop(0)


if __name__ == '__main__':
    main()

```

服务器差别也很小，只是在 RPC 方法的输入和输出分别变成了迭代器和生成器，没有其它区别。同时为了体现输入输出不是一对一的效果，我们按 50% 的概率对请求予以响应。

**实现客户端**

```
# client.py
import grpc

import pi_pb2
import pi_pb2_grpc


def generate_request():
    for i in range(1, 1000):
        yield pi_pb2.PiRequest(n=i)


def main():
    channel = grpc.insecure_channel('localhost:8080')
    client = pi_pb2_grpc.PiCalculatorStub(channel)
    response_iterator = client.Calc(generate_request())
    for response in response_iterator:
        print "pi(%d) =" % response.n, response.value


if __name__ == '__main__':
    main()

```

客户端的差别在于 RPC 的输入参数是一个生成器，输出是一个迭代器，这个跟服务器正好对应起来。

**运行**

```
# 开一个窗口
python server.py
# 再开一个窗口
python client.py

```

观察到输出如下

```
...
pi(981) = 3.14126849241
pi(982) = 3.14126882219
pi(984) = 3.14126947975
pi(985) = 3.14126980753
pi(986) = 3.14127013464
pi(987) = 3.1412704611
pi(988) = 3.14127078689
pi(989) = 3.14127111202
pi(991) = 3.14127176033
pi(992) = 3.1412720835

```

跑起来跟普通的 RPC 似乎没有太多区别，只不过从输出上看 n 的值并不连续，有很多请求被跳过了。

gRPC 除了支持客户端服务器双向 Streaming 之外，还可以单向 Streaming。如单请求流响应或者流请求单响应。比如客户端向服务器发送了流请求，包含了一堆子请求，服务器对这些子请求进行汇总处理，通过一个单响应回馈给客户端，这就是流请求单响应的一个使用场景。再比如推荐系统在给用户计算推荐商品时，可能需要进行非常复杂的计算，而且是分布计算，一次只能吐出几个结果，整体耗时会很长。这时就可以通过 Streaming 给客户端响应，算出一个推一个，而不是等到完全计算完毕再给结果，这样可以显著提升用户体验，这就是单请求流响应的使用场景。

## 异常处理

RPC 服务还需要处理的一个重要问题就是异常。当 RPC 服务器逻辑抛出异常时，客户端应该可以正确捕获到。GRPC 对异常的处理就是在响应头帧里使用 Status 和 Status-Message 两个 header 来标识异常的 code 和原因。对应的代码就是

```
class PiCalculatorServicer(pi_pb2_grpc.PiCalculatorServicer):

    def Calc(self, request, ctx):
        if request.n <= 0:
            ctx.set_code(grpc.StatusCode.INVALID_ARGUMENT)  # 参数错误
            ctx.set_details("request number should be positive")  # 错误具体说明
            return pi_pb2.PiResponse()
        s = 0.0
        for i in range(request.n):
            s += 1.0/(2*i+1)/(2*i+1)
        return pi_pb2.PiResponse(value=math.sqrt(8*s))

```

gRPC 框架内置了很多错误 code，当我想要抛出异常时，就从中挑选一个合适的 code，再加上一个错误说明的字符串就可以了。注意即使发生错误了，也需要返回一个正常的空对象。

客户端在处理异常时用 try...catch 包裹起 RPC 调用，然后从异常对象中提取 code 和 details 两个字段就知道具体发生的错误原因了。

```
def main():
    channel = grpc.insecure_channel('localhost:8080')
    client = pi_pb2_grpc.PiCalculatorStub(channel)
    for i in range(2):
        try:
            print "pi(%d) =" % i, client.Calc(pi_pb2.PiRequest(n=i)).value
        except grpc.RpcError as e:
            print e.code(), e.details()

```

## 自定义异常

gRPC 框架内部内置了很多异常状态码，我从源码里将这些错误码都列了出来

```
class StatusCode(enum.Enum):
    # 默认正常
    OK = (_cygrpc.StatusCode.ok, 'ok')
    # RPC 调用取消
    CANCELLED = (_cygrpc.StatusCode.cancelled, 'cancelled')
    # 未知错误，一般是服务器业务逻辑抛出了一个异常
    UNKNOWN = (_cygrpc.StatusCode.unknown, 'unknown')
    # 参数错误
    INVALID_ARGUMENT = (_cygrpc.StatusCode.invalid_argument, 'invalid argument')
    # 调用超时
    DEADLINE_EXCEEDED = (_cygrpc.StatusCode.deadline_exceeded, 'deadline exceeded')
    # 资源没找到
    NOT_FOUND = (_cygrpc.StatusCode.not_found, 'not found')
    # 资源重复创建
    ALREADY_EXISTS = (_cygrpc.StatusCode.already_exists, 'already exists')
    # 权限不足
    PERMISSION_DENIED = (_cygrpc.StatusCode.permission_denied, 'permission denied')
    # 资源不够
    RESOURCE_EXHAUSTED = (_cygrpc.StatusCode.resource_exhausted, 'resource exhausted')
    # 预检失败
    FAILED_PRECONDITION = (_cygrpc.StatusCode.failed_precondition, 'failed precondition')
    # 调用中止
    ABORTED = (_cygrpc.StatusCode.aborted, 'aborted')
    # 超出资源既有范围，比如断点续传的 offset 值超出文件长度
    OUT_OF_RANGE = (_cygrpc.StatusCode.out_of_range, 'out of range')
    # 服务器未实现协议中定义的 Service 接口
    UNIMPLEMENTED = (_cygrpc.StatusCode.unimplemented, 'unimplemented')
    # 服务器内部错误
    INTERNAL = (_cygrpc.StatusCode.internal, 'internal')
    # 服务挂了
    UNAVAILABLE = (_cygrpc.StatusCode.unavailable, 'unavailable')
    # 数据丢失
    DATA_LOSS = (_cygrpc.StatusCode.data_loss, 'data loss')
    # 鉴权错误
    UNAUTHENTICATED = (_cygrpc.StatusCode.unauthenticated, 'unauthenticated')

```

这些错误码和 HTTP 的一系列状态码基本都能对上。有很多类型我们通常都用不上，不过这些类型都是协议定义的，必然有相应的使用场景。看到这里，也许你会问：如果我想自定义业务的错误码，该怎么做呢？

有两种方法，第一个就是在 details 字段着手，它是一个字符串。我们可以通过 details 来传递序列化的自定义错误对象。

```
ctx.details(json.dumps({"biz_err_code": 10086, "msg": "something happened"}))

```

第二种方法是通过 metadata，gRPC 的请求和响应头都支持传递 metadata 信息，它是一个具有多个字符串 key/value 的字典，你可以在 metadata 中添加任意附加信息。

```
ctx.send_initial_metadata([(key1, value1), (key2, value2)])

```

注意 key/value 必须是字符串，否则 gRPC 框架会抛出错误。我们从客户端捕获到的异常中就可以将 metadata 读取出来。

```
e.initial_metadata()

```

## 单向消息

还有一种重要的消息类型叫单向 oneway 消息，这类消息适合不重要的日志型数据，发向了服务器不要求有响应，即使消息丢失了也无所谓。gRPC 不支持这类消息，gRPC 强制要求输入和输出都必须有类型，如果不需要返回结果，也必须使用内置的 Empty 的消息类型作为返回。相比而言，Thrift 是支持 oneway 消息的，阿里的 SOFA RPC 也支持 oneway 消息。

## 压缩

客户端可以在 channel 的选项中指定压缩算法属性，在请求的头部会携带压缩算法的参数传递给服务器。可惜的是目前只支持客户端压缩，Python 版本的 gRPC 服务器版本没有提供压缩选项。

```
import grpc
from grpc._cython.cygrpc import CompressionAlgorithm
from grpc._cython.cygrpc import CompressionLevel

chan_ops = [('grpc.default_compression_algorithm', CompressionAlgorithm.gzip),
            ('grpc.grpc.default_compression_level', CompressionLevel.high)]

channel = grpc.insecure_channel('localhost:8080', chan_ops)

```

## 重试

gRPC 默认不支持重试，如果 RPC 调用遇到错误，会立即向上层抛出错误。重试与否只能由用户自己的业务代码来进行控制或者由拦截器来统一控制。

## 超时

gRPC 默认支持超时选项，当客户端发起请求时，可以携带参数 timeout 指定最长响应时间，如果 timeout 时间内，服务器还没有返回结果，客户端就会抛出超时异常。

```
client.Calc(pi_pb2.PiRequest(n=i), timeout=5)  # 5s 超时

```

## 拦截器

gRPC 在客户端和服务器都提供了拦截器选项，用户可以通过拦截器拦截请求和响应。比如客户端可以通过拦截器统一在请求头里面增加 metadata，服务器可以通过拦截器来跟踪 RPC 调用性能等。

## 分布式 gRPC 服务

gRPC 服务器虽然性能好，但是压力大了，单个进程单个机器还是不够，最终是必须要上分布式的。分布式解决方案和普通的 RPC 大同小异，也是需要一个配置数据库来存储服务列表信息，常用的这类数据库由 zk、etcd 和 consul 等。读者可以使用实战小节的分布式方法包装一下 gRPC 的代码，就可以打造出分布式 gRPC 服务了。

## 推荐学习资料

考虑到 gRPC 内部协议非常复杂，短短一篇文章很难彻底将 gRPC 讲的全面，这里给读者推荐几篇扩展文章，感兴趣的读者可以继续深入阅读

1.  [TiDB —— 深入理解 gRPC](https://user-gold-cdn.xitu.io/2018/6/5/163cd9862a1faec1)
2.  [gRPC 官方全套文档 (英文阅读)](https://github.com/grpc/grpc/tree/master/doc)
3.  [HTTP2.0 详解——gitbook](https://legacy.gitbook.com/book/ye11ow/http2-explained/details)
4.  [gGRPC 设计与实现](https://platformlab.stanford.edu/Seminar%20Talks/gRPC.pdf)
5.  [HTTP2.0 Demo](http://www.http2demo.io/)
6.  [gRPC Errors](http://avi.im/grpc-errors/)

## 小结

本节我们搞清楚了 gRPC 的原理，学会了如何使用 gRPC 来开发服务。下一节我们看看 Facebook 开源的 RPC 通信框架 Thrift，看看它和 gRPC 有多大区别。

## 作业

1.  使用 gRPC 完成一个简单的 Redis 缓存读写服务。
2.  给以上基于 gRPC 的服务器提供分布式服务发现功能。