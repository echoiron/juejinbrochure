# Web UI 之编写首页功能性代码

紧接着上一个小节，将整个逻辑整理出来后，又准备好了静态资源，本小节将带领大家编写首页的功能性代码。

## 界面重构-编写首页功能性代码之爬虫状态统计

爬虫状态的统计，有利于日常管理时观察平台上爬虫的运行情况，对于运行情况的分析提供了一定的信息基础。这里的爬虫状态指的是待启动/正在运行/已结束爬虫数，也就是页面中「爬虫运行信息一览」部分的的数据。

![](https://user-gold-cdn.xitu.io/2018/10/15/1667587dde8223de?w=1236&h=200&f=png&s=12718)

通过对 Jobs 类的源码跟进，可以知道记录 Pending 状态、运行状态的对象，它们都跟`self.root`有关，所以代码改为：

```

def status_nums(self, finishes):
    """获取当前不同状态的爬虫数
    :param finishes: 已行完毕的爬虫列表
    :return: pending/running/finished 状态爬虫数的列表 list
    """
    pends = [queue.list() for project, queue in self.root.poller.queues.items()]
    run_value = self.root.launcher.processes.values()  # 正在运行的爬虫列表
    pend = pends[0] if pends else []
    return list(map(len, [pend, run_value, finishes]))

def features(self):
    """ 爬虫统计数据 """
    finishes = self.root.launcher.finished
    pending, running, finished = status_nums(self, finishes) # 待启动/正在运行/已结束爬虫数

```

为了保持主要代码的逻辑清晰，在上方代码中定义了`status_nums`函数以获取当前不同状态的爬虫数，并以列表形式返回，接着只需要通过 Python 列表的特性即可轻松将其拆包。

## 界面重构-编写首页功能性代码之爬虫运行时间统计

统计信息中必不可少的就是平均值、最大值与最小值，对应的是首页中**爬虫平均运行时间**、**最短运行时间**、**最长运行时间**。

![](https://user-gold-cdn.xitu.io/2018/10/15/16675b7f9d504b59?w=1210&h=157&f=png&s=15649)

> 为了提高文章可读性并保持逻辑清晰，在文章中`+`代表`新增的代码`，`-`代表`删除的代码`

通过定义新的函数来保持主代码的逻辑清晰，在下面的代码基础上进行改动。

```
+from functools import reduce


+def run_time_stats(finishes):
    """爬虫运行时间统计
    :param finishes: 已行完毕的爬虫列表
    :return: average-平均时间，shortest-最短运行时间， longest-最长运行时间
    """
    runtime = [microsec_trunc(f.end_time - f.start_time) for f in finishes]  # 爬虫运行时间
    # 平均时间, 求和不能用sum, 采用reduce进行计算
    average = reduce(lambda x, y: x + y, runtime) / len(runtime) if runtime else "0:00:00"
    shortest = min(runtime) if runtime else "0:00:00"  # 最短运行时间
    longest = max(runtime) if runtime else "0:00:00"  # 最长运行时间
    return average, shortest, longest

def features(self):
    """ 爬虫统计数据 """
    finishes = self.root.launcher.finished
    pending, running, finished = status_nums(self, finishes) # 待启动/正在运行/已结束爬虫数
    + average, shortest, longest = run_time_stats(finishes) # 爬虫运行时间统计

```

将爬虫运行记录对象`finishes`交给`run_time_stats`，通过`reduce`对爬虫运行时长进行累加并求平均值，然后通过`min`和`max`取出其中的运行时长最小值、运行时长最大值。如果没有爬虫被调用过，则`finishes`中也不会有运行记录，这样的取值方式会出异常的。考虑到页面的友好交互且规避取空值的问题，如果记录为空给出默认时间格式`0:00:00`，并且通过三目运算的方式决定最后的返回结果。

## 界面重构-编写首页功能性代码之爬虫运行时长 Rank 榜

榜单是数据可视化中比较有意思的模块，它可以很直观的告诉你，哪些爬虫运行的时间是最长，或许会在榜单上发现一些异常的爬虫。

![](https://user-gold-cdn.xitu.io/2018/10/15/16675b902b1915fa?w=1745&h=237&f=png&s=16053)

完成榜单排行这个功能需要获取所有的爬虫运行时间，并计算出爬虫运行时长，再根据运行时长做排序与切片：

> 为了提高文章可读性并保持逻辑清晰，在文章中`+`代表`新增的代码`，`-`代表`删除的代码`

```
+from datetime import timedelta
+from datetime import datetime

+def microsec_trunc(timelike):
    if hasattr(timelike, 'microsecond'):
        ms = timelike.microsecond
    else:
        ms = timelike.microseconds
    return timelike - timedelta(microseconds=ms)

+def time_rank(self, index=0):
    """爬虫运行时间排行 从高到低
    :param index: 排行榜数量 为0时默认取全部数据
    :return: 按运行时长排序的排行榜数据
    """
    finished = self.root.launcher.finished
    # 由于dict的键不能重复，但时间作为键是必定会重复的，所以这里将列表的index位置与时间组成tuple作为dict的键
    tps = {(microsec_trunc(f.end_time - f.start_time), i): f.spider for i, f in enumerate(finished)}
    # rps格式 [{"time": time, "spider": spider}, {"time": time, "spider": spider}]
    result = [{"time": str(k[0]), "spider": tps[k]} for k in sorted(tps.keys(), reverse=True)]  # 已排序
    if len(result) == 0:
        result = [{"rank": 0, "spider": "nothing", "time": "00:00:00"}]
    rps = result if index == 0 else result[:index]  # 为0时取所有排行，为int则进行切片
    return rps

+def time_ranks_table(self, index=0):
    """爬虫运行时间排行 从高到低
    :param index: 排行榜数量
    :return: 符合排行榜数的<table>表格
    """
    tds = []
    rps = time_rank(self, index=index)
    for i, r in enumerate(rps):
        aps = "<tr><td><i>{key}</i></td><td>{spider}</td><td>{number}</td></tr>".format(
            key=i, spider=r.get("spider"), number=r.get("time"))
        tds.append(aps)
    table = "".join(tds) if len(tds) else ""
    return table

def features(self):
    """ 爬虫统计数据 """
    finishes = self.root.launcher.finished
    pending, running, finished = status_nums(self, finishes) # 待启动/正在运行/已结束爬虫数
    average, shortest, longest = run_time_stats(finishes) # 爬虫运行时间统计
    +ranks = time_ranks_table(self, index=10) # 爬虫运行时间排行

```

上方代码中`time_ranks_table`接收 index 参数，计算后的结果列表根据传入的 index 参数进行切片，这样就可以实现排行榜的排行长度选择，比如运行时长前 15 的爬虫运行记录或运行时长前 10 的爬虫运行记录。其中`microsec_trunc`方法是 Scrapyd 在`webwite.py`中使用的方法，此处笔者将其代码复制到`webtools.py`中。由于在 HTML 中，表格代码结构为：

```
<table>
    <tbody>
        <tr>
            <td></td>
        </tr>
    </tbody>
</table>

```

所以这里只需要将`<tr><td></td></tr>`格式的数据生成即可，所以将时间计算与样式生成拆成两个不同的函数`time_rank`和`time_ranks_table`。

## 界面重构 - 编写首页功能性代码之爬虫调用情况统计

平台上部署的爬虫数量很多，随着时间的推移，你无法从数以千计的爬虫运行记录中判断到底哪一些爬虫是部署后重来未被调用过的，哪一些爬虫是被调用过的甚至哪一个爬虫被调用的次数最多，它被调用了多少次。

![](https://user-gold-cdn.xitu.io/2018/10/15/16675e3b43be94b7?w=601&h=143&f=png&s=15195)

对于爬虫是否被调用，也是需要根据已有条件进行计算或比对的，这里的逻辑是：

*   先获取所有项目中的所有爬虫名
*   再获取已运行结束的爬虫记录中的爬虫名
*   通过 set 的 different 进行比对，求出差值
*   通过 Counter 和 most\_common 计算得出被调用次数最多的爬虫及对应次数

> 为了提高文章可读性并保持逻辑清晰，在文章中`+`代表`新增的代码`，`-`代表`删除的代码`

```
+from collections import Counter


+def get_ps(self):
    """获取项目列表与爬虫列表
    :param self:
    :return: projects-项目列表， spiders-爬虫列表
    """
    projects = list(self.root.scheduler.list_projects())
    spiders = get_spiders(projects)
    return projects, spiders

+def get_invokes(finishes, spiders):
    """获取已被调用与未被调用的爬虫名称
    :param: finishes 已运行完毕的爬虫列表
    :param: spiders 爬虫列表
    :return: invoked-被调用过的爬虫集合, un_invoked-未被调用的爬虫集合, most_record-被调用次数最多的爬虫名与次数
    """
    invoked = set(i.spider for i in finishes)
    un_invoked = set(spiders).difference(invoked)
    invoked_record = Counter(i.spider for i in finishes)
    most_record = invoked_record.most_common(1)[0] if invoked_record else ("nothing", 0)
    return invoked, un_invoked, most_record

def features(self):
    """ 爬虫统计数据 """
    finishes = self.root.launcher.finished
    pending, running, finished = status_nums(self, finishes) # 待启动/正在运行/已结束爬虫数
    average, shortest, longest = run_time_stats(finishes) # 爬虫运行时间统计
    ranks = time_ranks_table(self, index=10) # 爬虫运行时间排行
    +projects, spiders = get_ps(self) # 项目/爬虫列表
    +invoked_spider, un_invoked_spider, most_record = get_invokes(finishes, spiders) # 被调用过/未被调用的爬虫

```

## 界面重构 - 编写首页功能性代码之爬虫项目及对应爬虫名和数量

Scrapyd 首页默认显示当前已部署的爬虫项目名称，当需要通过 API：

```
$ curl http://localhost:6800/schedule.json -p project -p spider

```

调用爬虫的时候，你可能无法完全记得到底哪个爬虫会在哪个项目里，你还需要通过另一个 API 去查看指定项目名中的爬虫，这显然是不友好的。所以 ScrapydArt 设计的时候，就将这个功能考虑了进来。

![](https://user-gold-cdn.xitu.io/2018/10/15/16675e539925bbe7?w=1743&h=121&f=png&s=9495)

这里可以回想之前对 Home 与 Jobs 的调试经验，可以通过`utils.py`中的`get_spider_list`方法获取指定项目名的爬虫名，这里用列表推导式将爬虫名依次传入`get_spider_list`，并将每一次得到的结果都保存到列表中：

> 为了提高文章可读性并保持逻辑清晰，在文章中`+`代表`新增的代码`，`-`代表`删除的代码`

```
+def get_psn(projects):
    """ 获取项目与对应爬虫的名称及数量
    :param projects: 项目列表
    :return: [{"project": i, "spider": "tip, sms", "number": 2}, {……}] 结果列表
    """
    return [{"project": i, "spider": ",".join(get_spider_list(i)), "number": len(get_spider_list(i))} for i in projects]

def features(self):
    """ 爬虫统计数据 """
    finishes = self.root.launcher.finished
    pending, running, finished = status_nums(self, finishes) # 待启动/正在运行/已结束爬虫数
    average, shortest, longest = run_time_stats(finishes) # 爬虫运行时间统计
    ranks = time_ranks_table(self, index=10) # 爬虫运行时间排行
    +projects, spiders = get_ps(self) # 项目/爬虫列表

```

将上一步得到的 projects 作为参数传入 get\_ps 函数中对所需的数据进行计算，获得`[{"project": i, "spider": "tip, sms", "number": 2}, {……}]`格式的结果列表。

## 界面重构 - 编写首页功能性代码之项目总数与爬虫总数

我们需要清楚的知道平台上到底部署了多少个项目，总共有多少个爬虫，这信息跟你服务器内存承受的压力息息相关。

> 假如你有 30 多个爬虫，但是服务器只有 4G 内存，当你的爬虫火力全开时服务器有可能承受不住。

![](https://user-gold-cdn.xitu.io/2018/10/15/166766ba802c5698?w=474&h=349&f=png&s=11063)

ScrapydArt 就为你计算了这些数据。上方的代码中通过`get_ps`函数已经将数值计算好，并且在`features`函数中进行了拆包。

## 小结

将所有的数据处理函数都放到`webtools.py`中有利于后期维护与管理，在需要使用的地方导入并调用即可。