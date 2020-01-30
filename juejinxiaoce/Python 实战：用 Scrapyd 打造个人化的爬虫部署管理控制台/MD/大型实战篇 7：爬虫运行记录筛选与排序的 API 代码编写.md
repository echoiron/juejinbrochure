# 爬虫运行记录筛选与排序的 API 代码编写

我们在上一个小节编写了几个常用的 API ，并且它们的渲染方法都是 render\_GET。接下来让我们来尝试使用 render\_POST 方法编写功能更为复杂的 API 。

## 编写 API-爬虫运行记录筛选过滤功能(POST)

> 功能需求场景

爬虫已经运行了几个月，积累了超过 10000 条运行记录。最近你发现爬虫`tips`的数据结果异常，你希望通过查看日志来查找原因，但在 10000 条记录中依次寻找`tips`的记录是不现实的，这个时候你需要一个过滤功能，帮助你将`tips`爬虫的日志找出来。

> 实现逻辑

在此之前的几个 API 都是基于 website.py 改造时编写的功能做的，并且都是 GET 请求方式，下面的 API 选择了 POST 的请求方式。

由于筛选过滤功能条件较多，所以整个 API 实现过程比较抽象。

这里先来看这个 API 所需要实现的功能：

*   支持按`时间范围`对爬虫运行记录进行筛选过滤
*   支持按`爬虫名称`对爬虫运行记录进行筛选过滤
*   支持按`项目名称`对爬虫运行记录进行筛选过滤
*   支持按`运行时长`对爬虫运行记录进行筛选过滤
*   可指定结果集长度

对于可指定结果集长度，上面的 API 已经通过传入 index 参数结合切片实现了，那么要按照几个不同类型的属性对爬虫运行记录进行筛选过滤应该怎么设计呢？

*   首先应该想到的是，通过一个参数来区分这集中不同的属性
*   然后根据区分出来的属性，设定不同的参数
*   再在具体的方法中校验参数
*   如果参数通过校验则进行处理并返回结果集，如果不通过则返回提示信息

有一点思路了，但不是很清晰，那我们来画个图：

![](https://user-gold-cdn.xitu.io/2018/10/15/1667723c790c7a9b?w=1424&h=1096&f=png&s=72333)

有了流程图，再配合设计思路，就可以开始编码了。

首先编写类的基本结构：

```
class Filter(WsResource):
    @decorator_auth
    def render_POST(self, request):
        pass

```

然后拿到所有的爬虫运行记录对象：

```
def render_POST(self, request):
        +finishes = self.root.launcher.finished

```

接着定义允许的几种过滤类型及对应的方法：

```
    +def _filter_time(self):
        pass

    +def _filter_project(self):
        pass

    +def _filter_spider(self):
        pass

    +def _filter_runtime(self):
        pass
        
    def render_POST(self, request):
            finishes = self.root.launcher.finished
            +types_func = {"time": self._filter_time, "project": self._filter_project,
            "spider": self._filter_spider, "runtime": self._filter_runtime}

```

对于传入的这些参数，是需要校验的，比如传入的 index 值必须为 int 类型，在 webtools.py 中定义一个函数：

```
def valid_index(request, arg, ins=None, default=0):
    _ins = {"str": str, "int": int, "float": float, "list": list, "tuple": tuple, "dict": dict, "set": set}
    if not isinstance(arg, str):
        return default
    args = native_stringify_dict(copy(request.args), keys_only=False)
    try:
        value = args[arg][0]
        if ins not in _ins.keys():
            return default
        ins_type = _ins.get(ins)
        if not isinstance(value, ins_type):
            try:
                return ins_type(value)
            except Exception as err:
                logging.info(err)
                return default
        return default
    except:
        return default

```

*   先申明允许传入的参数类型
*   通过 isinstance 过滤掉不为 str 的参数值
*   使用 Scrapyd 中原有的 `native_stringify_dict` 获取 arg 集合
*   判断 ins 是否在预先声明的参数类型中，如果不在则将值返回，如果在则往下执行
*   获取参数类型并复制给 ins\_type
*   判断 value 的类型是否为取到的 `ins_type`，如果不是则返回默认值 10，如果不是则尝试返回 ins\_type(value)，假如产生异常则返回默认值。

使用这个函数，可以对传入参数及传入的参数数据类型做校验，可能你还有点迷茫，请先继续往下看。此时在 webservice.py 中引入 valid\_index

```
# webservice&#46;py
from .webtools import valid_index

```

这里要对传入的 index 进行校验：

```
index = int(valid_index(request=request, arg="index", ins="int", default=10))

```

使用时指定 arg 为 index，并且将数据类型设置为 int，再为其设置默认值。这个时候，再来看 valid\_index 就清晰一些了：

*   首先校验`arg`是否为`str`，这里的 arg 值为`index`那自然是通过的，进入下一步
*   第二步取出 request 请求时携带的参数
*   第三步尝试取出请求参数中的`index`参数的值，赋值给 value
*   接下来判断指定的数据类型`int`是否在预设的数据类型范围内
*   然后从预设数据类型范围内取出数据类型
*   判断 value 的数据类型是否是指定的`int`类型
*   最后将其类型进行强制转换，这里可以理解为 int(value)，确保了返回的值一定是指定的数据类型。
*   其中只要任何一个地方无法满足判断条件，都会将 default 返回

> 对于参数的校验

上面完成了对于请求时携带的参数类型校验，那么还有其他的一些校验呢？比如`按时间范围`筛选需要提交的参数为`起始时间`与`结束时间`，而`按爬虫名称`筛选则需要传入`爬虫名称`即可。对于这样多变的参数，应该如何校验和取值呢？

有了上面那个校验方法的编写经验，我们知道可以通过**预设**来进行限定。在 webtools.py 中新建函数：

```
def valid_args(request, arg, default={}):
    _types = {"time": ["st", "et"], "project": ["project"], "spider": ["spider"],
              "runtime": ["runtime"], "order": ["order"]}
    if isinstance(arg, str):
        args = native_stringify_dict(copy(request.args), keys_only=False)
        try:
            type_name = args[arg][0]
            if type_name not in _types.keys():
                return default
            try:
                values = _types.get(type_name)
                params = [args[i][0] for i in values]
                res = dict(zip(values, params))
                res["type"] = type_name
                return res
            except Exception as err:
                logging.info(err)
                return default
        except Exception as err:
            logging.info(err)
            return default
    return default

```

暂且先不用理解它的逻辑，在使用的时候再带着具体的参数过来一步步理解。

到 webservice.py 中将其引入：

```
# webservice&#46;py
from .webtools import valid_args

```

然后在 Filter 类中使用它：

```
filter_params = valid_args(request=request, arg="type")

```

> 带着参数回顾 valid\_args

调用`valid_args`函数，传入的`arg`参数为`"type"`，将结果赋值给 `filter_params`，现在进入`valid_args`中，看看它的运行流程：

*   首先校验传递的 arg 参数值类型，如果为 str 则向下执行
*   尝试从 request 请求参数中取出`type`参数对应的值
*   判断值是否在预设的类型中，即`_types`的 keys()，如果在则向下执行
*   根据类型值从预设字典中取出对应的参数，如"time": \["st", "et"\]中 time 为类型，"st"和"et"为对应参数。
*   再从 request 中取出对应参数的值，并将类型与参数值生成新的字典并返回
*   其中任何一处未通过判断，或者执行期间产生异常，都将默认值返回

拿到返回值后如何使用它呢？在 Filter 类的 render\_POST 中继续增加代码：

```
filter_type = filter_params.get("type")
if filter_type in types_func.keys():
    funcs = types_func.get(filter_type)
    if funcs:
        spiders = funcs(filter_params=filter_params, finishes=finishes, request=request, index=index)
        return {"node_name": self.root.nodename, "status": "ok", "spider": spiders}
return {"node_name": self.root.nodename, "status": "error", "message": "query parameter error"}

```

*   从返回值中取出`type`并判断其是否在预设的类型中，如果在则取出预设类型对应的处理方法
*   将所需的参数传递给预设类型对应的处理方法（此时并不确定具体是哪个方法，所以参数在设计时要考虑周全，笔者在设计时由于需求的不同，反复的修改过参数）。
*   方法执行完毕后将结果返回，这边用`spiders`变量接收
*   最后将结果以 json 格式返回，交给页面渲染
*   如果在类型取值时无法取到值，则返回参数错误的提示

> 具体的筛选过滤方法

预设中制定了几个具体的方法，它们根据传入的参数对爬虫运行记录进行筛选过滤，那么它们是如何设计和处理的呢？

```
def _filter_time(self, filter_params, finishes, request, index, default=[]):
    if not valid_params(filter_params):
            return default
    res = valid_date(filter_params.get("st"), filter_params.get("et"))
    if not res:
        return default
    st, et = res
    spiders = [spider_dict(i) for i in finishes if (i.start_time - st).days >= 0 and (et - i.start_time).days >= 0][:index]
    return spiders

```

这里以`_filter_time`方法为例，它对应的过滤类型是`time`即`按时间范围`筛选。

*   首先它需要校验传入的 `filter_params` 类型与长度，这里通过`valid_params`实现，`valid_params`代码为：

```
def valid_params(filter_params):
    if not isinstance(filter_params, dict):
        return False
    if not len(filter_params):
        return False
    return True

```

返回布尔值。

*   而后取出`st`和`et`的值并进行校验，主要是确保起始时间与结束时间的时间格式，然后确保结束时间要比起始时间的值要大，所以对于时间的校验也需要在 webtools.py 中新增时间校验方法 valid\_date:

```
def valid_date(start, end, half="%Y-%m-%d %X"):
    """ 校验时间格式，如果错误则返回False，如果正确则返回格式化后的时间字符串 """
    try:
        start_time, end_time = datetime.strptime(start, half), datetime.strptime(end, half)
        days = (end_time - start_time).days
        if days < 0:
            return False
        return start_time, end_time
    except Exception as err:
        logging.info(err)
    return False

```

*   传入的参数类型及正确性通过校验后，即可在爬虫运行记录中进行筛选。已知爬虫运行记录是一个列表对象，所以这里可以通过 for 循环的方式完成记录的筛选、切片与重组。

```
spider_list = []
for i in finishes:
    if (i.start_time - st).days >= 0 and (et - i.start_time).days >= 0
    spider_list.append(spider_dict(i))
spiders = spider_list[:index]

```

对于 for 循环，笔者习惯于用列表推导式的方式，可以很简短的完成：

```
spiders = [spider_dict(i) for i in finishes if (i.start_time - st).days >= 0 and (et - i.start_time).days >= 0][:index]

```

## 你的考验

上面通过需求分析、代码逻辑整理、绘制流程图以及代码编写及运行流程分析完成了Filter大部分功能的编写，并且以 `_filter_time` 作为例子，演示了如何完成爬虫运行记录的过滤筛选。编程光看还不够，还需要动脑思考、动手实践，所以剩下的`_filter_project`、`_filter_spider`、`_filter_runtime`就交给你来实现。

记得，编写完成后要到配置文件中为 Filter 添加路由映射

自然，笔者已经编写好对应的代码: [Filter](https://github.com/dequinns/ScrapydArt/blob/master/scrapydart/webservice.py)，但笔者希望你亲身实践过后，再来看这部分的代码，这对你更有帮助。

## Filter 筛选效果演示

首先，在编写完代码后启动 ScrapydArt，通过 Postman 工具向 ScrapydArt 上的爬虫发起调用。基于爬虫运行的结果记录，再做测试：

![](https://user-gold-cdn.xitu.io/2018/10/16/1667abe43f8be63f?w=1740&h=626&f=png&s=119992)

上图为当前爬虫运行记录，共有 12条，时间范围在`2018-10-16 10:33:12`~`2018-10-16 10:34:17`之间，下面的测试都将基于此记录。

> 根据指定的时间范围进行过滤

这里笔者使用 Postman 工具对 Filter 进行筛选，可以直接在 URL 栏输入所有的请求参数：

```
http://localhost:6800/filter.json?un=qu&pwd=qus7&type=time&st=2018-10-16%2010:33:00&et=2018-10-16%2010:33:45

```

（备注：按时间范围过滤代码在编写时，指定的判断元素为 `start_time`，所以结果集也是根据爬虫的 `start_time` 进行过滤后得到的）

![](https://user-gold-cdn.xitu.io/2018/10/16/1667ac4f518bdfd1?w=1019&h=736&f=gif&s=154857)

得到的结果为：

```
{
    "node_name": "gannicus-PC",
    "status": "ok",
    "spider": [
        {
            "start_time": "2018-10-16 10:33:12",
            "end_time": "2018-10-16 10:33:30",
            "runtime": "0:00:17.848460",
            "project": "arts",
            "spider": "tips",
            "logs": "logs/arts/tips/d1edca42d0eb11e8825c54ee75c0e204.log"
        },
        {
            "start_time": "2018-10-16 10:33:27",
            "end_time": "2018-10-16 10:33:34",
            "runtime": "0:00:07.225114",
            "project": "Fabias",
            "spider": "fabias",
            "logs": "logs/Fabias/fabias/d9f34884d0eb11e8825c54ee75c0e204.log"
        },
        {
            "start_time": "2018-10-16 10:33:32",
            "end_time": "2018-10-16 10:33:39",
            "runtime": "0:00:07.151288",
            "project": "Fabias",
            "spider": "fabias",
            "logs": "logs/Fabias/fabias/da66a54ad0eb11e8825c54ee75c0e204.log"
        },
        {
            "start_time": "2018-10-16 10:33:37",
            "end_time": "2018-10-16 10:33:58",
            "runtime": "0:00:21.086578",
            "project": "Fabias",
            "spider": "players",
            "logs": "logs/Fabias/players/dde33972d0eb11e8825c54ee75c0e204.log"
        },
        {
            "start_time": "2018-10-16 10:33:42",
            "end_time": "2018-10-16 10:34:04",
            "runtime": "0:00:21.447053",
            "project": "Fabias",
            "spider": "players",
            "logs": "logs/Fabias/players/de740a2ed0eb11e8825c54ee75c0e204.log"
        }
    ]
}

```

总记录有 12 条，这里按时间范围筛选出来的结果数量为 5，从数量上看应该是没有出问题的。那么来仔细的比对一下每条记录中的`start_time`是否在指定的时间范围内？

确认了一下，这 5 条记录中的`strat_time`从`2018-10-16 10:33:12`到`2018-10-16 10:33:42`，全都是属于指定的范围内，说明这个功能已经可以正常使用了。

> 根据指定的项目名称进行过滤

请求的 URL 为：

```
http://localhost:6800/filter.json?un=qu&pwd=qus7&type=project&project=arts

http://localhost:6800/filter.json?un=qu&pwd=qus7&type=project&project=Fabias

```

笔者将用这两个不同的项目名称发出请求：

![](https://user-gold-cdn.xitu.io/2018/10/16/1667aca84676c4ad?w=1022&h=764&f=gif&s=272771)

从 GIF 动图中可以看到，返回的结果集也是符合要求的，会根据指定的`project`名称筛选出对应的爬虫记录。

> 根据指定的爬虫名称进行过滤

请求的 URL 为：

```
http://localhost:6800/filter.json?un=qu&pwd=qus7&type=spider&spider=tips

http://localhost:6800/filter.json?un=qu&pwd=qus7&type=spider&spider=quinns

```

笔者将用这两个不同的爬虫名称发出请求

![](https://user-gold-cdn.xitu.io/2018/10/16/1667acd6193734f3?w=1024&h=762&f=gif&s=319217)

从 GIF 动图中可以看到，返回的结果集也是符合要求的，会根据指定的`spider`名称筛选出对应的爬虫记录。

> 根据指定的运行时长进行筛选

请求的 URL 为：

```
http://localhost:6800/filter.json?un=qu&pwd=qus7&type=runtime&runtime=5&compare=less

http://localhost:6800/filter.json?un=qu&pwd=qus7&type=runtime&runtime=5&compare=greater

```

其中的 less 代表小于，即筛选运行时长小于 5秒的记录;greater 代表大于，即筛选运行时长大于 5秒的记录。

笔者将用这两个不同的 URL 发出请求

![](https://user-gold-cdn.xitu.io/2018/10/16/1667ad6f8f42ca80?w=1016&h=757&f=gif&s=342540)

从 GIF 动图中可以看到，返回的结果集也是符合要求的，会根据指定的`compare`参数筛选出对应的爬虫记录。

## 编写 API-爬虫运行记录排序(POST)

这个排序的功能就很简单明了：`可选升降序`、`可指定排序结果长度`、`可指定排序元素`。这里列出实现逻辑：

*   从 request 中获取请求参数
*   对请求参数进行校验
*   预设不同参数的处理方法
*   将结果集按照 index 进行切片 其中的获取请求参数、校验、预设、切片都已经在编写 Filter 的时候完成了，可以直接拿来使用，所以整个 Order 的代码为：

```
class Order(WsResource):

    @decorator_auth
    def render_POST(self, request):
        """ 根据传入的参数不同，对爬虫运行记录进行排序
        目前支持按启动时间、按结束时间、按项目名称、按爬虫名称以及运行时长进行筛选过滤，index默认10
        计算并取出排序对应参数以及index参数，并对其进行校验
        根据index进行切片
        :return 符合名称的爬虫信息列表 [{}, {}]
        """
        finishes = self.root.launcher.finished
        orders = {
            "start_time": lambda x: x.start_time, "end_time": lambda x: x.end_time,
            "spider": lambda x: x.spider, "project": lambda x: x.project,
            "runtime": lambda x: x.end_time-x.start_time
        }
        index = int(valid_index(request=request, arg="index", ins="int", default=10))
        params = valid_args(request=request, arg="type")
        order_key = params.get(params.get("type"))
        reverse_param = int(valid_index(request=request, arg="reverse", ins="int", default=0))
        if order_key not in orders.keys() or reverse_param not in [0, 1]:
            return {"node_name": self.root.nodename, "status": "error", "message": "query parameter error"}
        ordered = sorted(finishes, key=orders.get(order_key), reverse=reverse_param)
        spiders = [spider_dict(i) for i in ordered[:index]]
        return {"node_name": self.root.nodename, "status": "ok", "spider": spiders}

```

代码编写完后记得到配置文件中添加路由映射。

代码的执行流程与 Filter 也非常相似，这里就留给大家做考验，如果有不明白的地方，可以在留言处提出，笔者会为大家解答的。

> 排序测试

按运行时长进行排序(默认)：

```
http://localhost:6800/order.json?un=qu&pwd=qus7&type=order&order=runtime

```

![](https://user-gold-cdn.xitu.io/2018/10/16/1667aedcf9cad95d?w=1025&h=758&f=gif&s=554958)

按启动时间进行排序(默认->降序)：

```
http://localhost:6800/order.json?un=qu&pwd=qus7&type=order&order=start_time

http://localhost:6800/order.json?un=qu&pwd=qus7&type=order&order=start_time&reverse=1

```

![](https://user-gold-cdn.xitu.io/2018/10/16/1667af5c693deedb?w=1022&h=759&f=gif&s=445804)

GIF 动图中展示了升序与降序两种排序方式的请求过程。 还可以指定其他的排序元素，这里就不一一演示了，大家可以在编写完代码后自己逐一测试，思考一下有没有更好的解决办法或者写法。