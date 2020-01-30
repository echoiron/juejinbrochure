# 协议 3：Protobuf 二进制协议结构

![](https://user-gold-cdn.xitu.io/2018/5/12/16351d7015b44acb?w=660&h=400&f=png&s=33303)

Protobuf 协议是 Google 开源的二进制 RPC 通讯协议，它可能是互联网开源项目中使用最为广泛的 RPC 协议。

Protobuf 提供了一种描述通讯协议的接口描述语言 IDL，通过编写接口协议，Protobuf 可以自动生成多种语言的 RPC 通讯代码，目前官方已经支持了近 10 种语言。

![](https://user-gold-cdn.xitu.io/2018/5/12/16351d85da46fab1?w=2152&h=374&f=png&s=143778)

如上图所示，最左边使用 Protobuf 的 IDL 语言编写的协议接口。使用 protoc 工具可以直接根据协议生成 RPC 通讯的所有代码。右边分别是 Java 语言和 C++ 语言直接使用 protoc 生成的代码进行 RPC 通讯的例子。

## 协议格式

**键值对**

![](https://user-gold-cdn.xitu.io/2018/5/31/163b4f7aa45e3255?w=818&h=68&f=png&s=9614)

Protobuf 传输的是一系列的键值对，如果连续的键重复了，那说明传输的值是一个列表 (repeated)。图上的 key3 就是一个列表类型 (repeated)。

键 **key** 两部分组成：**tag** 和 **type**。

*   **tag**

Protobuf 将对象中的每个字段和正数序列 (tag) 对应起来，对应关系的信息是由生成的代码来保证的。在序列化的时候用整数值来代替字段名称，于是传输流量就可以大幅缩减。如果字段较少，它就使用 4 个 bits 来表示，最多支持 16 个字段。如果字段数量超过 16 个，那就再加 1 个字节，如果还不够那就再加 1 个字节。你也许猜到了，这个 tag 值使用的是 varint 编码。理论上字段的长度不设上限，因为 varint 可以通过扩展字节数支持任意大的非负整数。

*   **type**

Protobuf 将字段类型也和正数序列 (type) 对应起来，每一种原生的 java 类型都唯一对应一个正数，类型信息也非常节省。type 使用 3 个 bits 表示，最多支持 8 种类型。

也许你会认为 8 种类型怎么够呢？放心，肯定够的！因为一个 zigzag 类型可以表示所有的类整数类型，byte/short/int/long/bool/enum/unsigned byte/unsigned short/unsigned int/unsigned long 这些类型都可以使用 zigzag 表示。而 Python 语言的整数更加特别，它根本就不区分整数的位数，甚至可以是一个 BigInteger。varint 的特长就在于此，它可以无限扩展位数大小，可以表示无限的整数值。而字节数组、字符串和嵌套对象都可以使用一种称之为 length 前缀 (length-delimited) 的类型来表示。另外 float 和 double 类型各占一个类型。最终你看，连 8 个类型都没有使用到。

**key = tag || type**

![](https://user-gold-cdn.xitu.io/2018/5/12/1635319e4749b89a?w=1306&h=300&f=png&s=33939)

Protobuf 将字段名称对应的整数 (tag) 和字段类型对应的整数 (type) 合并在一起凑成一个字节。如果一个对象的字段比较多，对应的正数比较大，那么就使用两个字节，如果还不够，那就三个字节。无所谓已经说过了，这里使用的是 varint 编码，不管你字段有多少，都可以支持。最高位的 0 和 1 是标志是 1 个字节还是 2 个字节，它是 varint 编码的扩展位。

下面我们看 value 部分，value 部分随着类型 type 的不同而具有不同的形式

**整数**

Protobuf 的整数数值使用 zigzag 编码。zigzag 编码支持负数值，varint 编码的都是非负数。这个在第三节已经讲过了，它们都是变长整数。

**浮点数** 浮点数分为 float 和 double，它们分别使用 4 个字节和 6 个字节序列化，这两个类型的 value 没有做什么特殊处理，它就是标准的浮点数。

**字符串**

![](https://user-gold-cdn.xitu.io/2018/5/31/163b4ff98c9e7aa9?w=684&h=101&f=png&s=8121)

Protobuf 的字符串值使用长度前缀编码。第一个字节是字符串的长度，后面相应长度的字节串就是字符串的内容。如果字符串长度很长，那么长度前缀就不止一个字节，它可能是两字节三字节四字节，你也许猜到了，这个长度采用的是 varint 编码。

**嵌套**

Protobuf 支持类型嵌套。嵌套类型的 type 同字符串的 type 一样，都是 length 前缀。第一个字节 (varint) 表示字节长度，后面相应长度的字节串就是嵌套对象的整个内容，这部分内容会递归使用 Protobuf 进行编码解码。

**可选类型**

Protobuf 支持可选类型。不过二进制流里面可没有使用任何标志为来表示字段是否可选。它只是在运行时做了检查，如果一个必须的字段的 tag 在二进制流里面没有出现，那就会抛出一个运行时异常。当Protobuf升级到3.0的时候，可选类型消失了，取而代之的是所有的类型都是可选类型。也就是说发送端即使没有填充该字段，接收端也不会抛出错误了，字段可选与否完全依赖于用户自己去检查了。

## 一个简单的例子

```
message Person {
    required string user_name        = 1;  // 必须字段
    optional int64  favourite_number = 2;  // 可选字段
    repeated string interests        = 3;  // 列表类型
}

```

```
var person = new Person{
    user_name: "Martin",
    favourite_number: 1337,
    interests: ["daydreaming", "hacking"]
}

```

![](https://user-gold-cdn.xitu.io/2018/5/12/163523f1ca7bd3bc?w=2292&h=958&f=png&s=79609)

上图的二进制序列就是一个完整的 person 对象的序列化数据，包含了字符串、整数和数组 (列表) 类型。仔细观察这张图，我们可以看到每个字段都有一个共同的 tag|type 前缀。如果是类整数，后面就直接跟 value。如果是 length-delimeted 类型，后面就先跟一个长度，再跟一个字节串。因为整数采用 zigzag 编码，所以 Protobuf 对整数的 bits 要进行一些转换 (图中的整数 1337)。

![](https://user-gold-cdn.xitu.io/2018/6/4/163c8b1ad93b2165?w=622&h=222&f=png&s=19476)

## 消息边界

*   Protobuf 并没有定义消息边界，也就是没有消息头。消息头一般由用户自己定义，通常使用长度前缀法来定义边界。
*   同样 Protobuf 也没有定义消息类型，当服务器收到一串消息时，它必须知道对应的类型，然后选择相应消息类型的解码器来读取消息。这个类型信息也必须由用户自己在消息头里面定义。

## 小结

Redis 的协议和 Protobuf 的协议我们都分析完了，但是它们真的如看上面介绍的那样很完美么？在实战开发中，会有哪些需要注意的坑呢？

下一节我们将从一个有意思的角度来分析 Redis 的协议为什么有问题。

## 作业

Thrift 是 Faceook 开源的另一款开源 RPC 框架，它的协议和 Protobuf 非常类似，请读者搜寻相关文档了解 Thrift 的结构编码，看看它与 Protobuf 有怎样的不同？

*   [Avro, Protobuf 和 Thrift 协议的异同](https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html)