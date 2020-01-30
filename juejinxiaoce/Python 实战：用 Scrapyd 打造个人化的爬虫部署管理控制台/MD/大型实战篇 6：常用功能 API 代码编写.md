# 常用功能 API 代码编写

本小节我们使用 render\_GET 方法来将首页已有的功能编写成 API ，并且在首页已有功能的基础上逐步加大难度，将 API 的功能变得更丰富。

## 编写 API-爬虫调用情况(GET)

已知`get_ps`方法可以获取项目名列表和爬虫名列表，且向`get_invokes`方法传入已结束的爬虫运行记录与爬虫名列表，可以计算出被调用、未被调用与调用次数最多的爬虫及对应次数。

```
class ScheduleList(WsResource):
    @decorator_auth
    def render_GET(self, request):
        """爬虫调用情况
        如被调用过的爬虫、未被调用过的爬虫、被调用次数最多的爬虫名称及次数
        """
        finishes = self.root.launcher.finished
        projects, spiders = get_ps(self)  # 项目/爬虫列表
        invoked_spider, un_invoked_spider, most_record = get_invokes(finishes, spiders)  # 被调用过/未被调用的爬虫
        return {"node_name": self.root.nodename, "status": "ok", "invoked_spider": list(invoked_spider),
        "un_invoked_spider": list(un_invoked_spider), "most_record": most_record}

```

这样就完成了第一个 API，到配置文件中的`service`级配置下为其增加路由映射关系：

```
schedulelist.json = scrapyd.webservice.ScheduleList

```

然后使用浏览器或 Postman 工具对 API 进行测试，演示中笔者使用浏览器：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676e3503a220fa?w=1708&h=914&f=gif&s=604375)

从演示动图中可以看到，API 已经可以正常使用了，并且返回了正确的数据:

```
{"node_name": "gannicus-PC", "status": "ok", "invoked_spider": ["tips", "fabias"], "un_invoked_spider": ["players", "quinns", "flu_event", "flu_teams"], "most_record": ["fabias", 2]}


```

## 编写 API-爬虫运行时长统计(GET)

有了 Web 界面重构时编写的功能代码，前面这一些 API 写起来比较轻松

```
class RunTimeStats(WsResource):
    @decorator_auth
    def render_GET(self, request):
        """爬虫运行时间统计
        如平均运行时长、最短运行时间、最长运行时间
        """
        finishes = self.root.launcher.finished
        average, shortest, longest = list(map(str, run_time_stats(finishes)))
        return {"node_name": self.root.nodename, "status": "ok", "average": average,
        "shortest": shortest, "longest": longest}

```

增加路由映射：

```
runtimestats.json = scrapyd.webservice.RunTimeStats

```

使用测试：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676e861d147de1?w=1713&h=634&f=gif&s=187900)

得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "average": "0:00:08.333333", "shortest": "0:00:07", "longest": "0:00:11"}

```

## 编写 API-项目及爬虫数量统计(GET)

依样画葫芦：

```
class PsnStats(WsResource):
    @decorator_auth
    def render_GET(self, request):
        """ 项目及爬虫数量统计,如项目总数、爬虫总数 """
        project_nums, spider_nums = list(map(len, get_ps(self)))
        node_name = self.root.nodename
        return {"node_name": node_name, "status": "ok", "project_nums": project_nums, "spider_nums": spider_nums}

```

增加路由映射：

```
psnstats.json = scrapyd.webservice.PsnStats

```

使用测试：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676ebfbf6dbe75?w=1711&h=300&f=gif&s=218142)

得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "project_nums": 2, "spider_nums": 6}

```

## 编写 API-项目与对应爬虫名及爬虫数量(GET)

照猫画虎：

```
class ProSpider(WsResource):
    @decorator_auth
    def render_GET(self, request):
        """ 项目与对应爬虫名及爬虫数量,如[{"project": i, "spider": "tip, sms", "number": 2}, {……}] """
        projects, spiders = get_ps(self)  # 项目/爬虫列表
        pro_spider = get_psn(projects)  # 项目与对应爬虫
        return {"node_name": self.root.nodename, "status": "ok", "pro_spider": pro_spider}

```

增加路由映射：

```
prospider.json = scrapyd.webservice.ProSpider

```

使用测试：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676ee6043022cd?w=1718&h=593&f=gif&s=166337)

得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "pro_spider": [{"project": "arts", "spider": "tips", "number": 1}, {"project": "Fabias", "spider": "fabias,flu_event,flu_teams,players,quinns", "number": 5}]}


```

## 编写API-爬虫运行时长排行 从高到低(GET)

```

class TimeRank(WsResource):
    """ 爬虫运行时长排行,根据index参数进行切片 """

    def time_rank(self, index=0):
        """爬虫运行时间排行 从高到低
        :param index: 排行榜数量 默认返回全部数据，index存在时则切片后返回
        :return: 按运行时长排序的排行榜数据 [{"time": time, "spider": spider}, {}]
        """
        finished = self.root.launcher.finished
        # 由于dict的键不能重复，但时间作为键是必定会重复的，所以这里将列表的index位置与时间组成tuple作为dict的键
        tps = {(microsec_trunc(f.end_time - f.start_time), i): f.spider for i, f in enumerate(finished)}
        result = [{"time": str(k[0]), "spider": tps[k]} for k in sorted(tps.keys(), reverse=True)]  # 已排序
        ranks = result if not index else result[:index]
        return ranks

    @decorator_auth
    def render_GET(self, request):
        index = int(valid_index(request=request, arg="index", ins="int", default=10))
        ranks = self.time_rank(index=index)  # 项目/爬虫列表
        return {"node_name": self.root.nodename, "status": "ok", "ranks": ranks}

```

**补充说明**：虽然之前的功能代码已有，但是这个 API 却完成了与之前不同的功能，爬虫运行时长排行的结果数量可以根据参数指定。设定默认的`index为 0`，`result`部分将所得结果通过`sorted 函数`进行排序，最后通过判断是否传入 index 来决定排行的长度。

增加路由映射：

```
timerank.json = scrapyd.webservice.TimeRank

```

使用测试：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676f7ace722e5d?w=1705&h=517&f=gif&s=423780)

当不传入 index 时得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "ranks": [{"time": "0:00:11", "spider": "tips"}, {"time": "0:00:07", "spider": "fabias"}, {"time": "0:00:07", "spider": "fabias"}]}

```

当指定 index=2 时得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "ranks": [{"time": "0:00:11", "spider": "tips"}, {"time": "0:00:07", "spider": "fabias"}]}

```

对比发现，默认排行长度此时为 3，但指定 index 后返回的结果中排行长度为 2，证明排行结果长度控制的测试也通过了。

> 你的任务：控制排行的升降序

类似 index 传递参数，可以指定升降序 reverse 为 True 或 False，从而实现升降序的控制，这个任务就交给你了！

## 编写 API-爬虫调用次数排行 从高到低(GET)

```
class InvokeRank(WsResource):
    """ 爬虫被调用次数排行 根据index参数进行切片 """

    def _invoke_rank(self, finishes, index=10):
        """获取爬虫被调用次数排行
        :param: finishes 已运行完毕的爬虫列表
        :param: spiders 爬虫列表
        :return: ranks-降序排序的爬虫调用次数列表 [("tip", 3), ()]
        """
        invoked_record = Counter(i.spider for i in finishes)
        ranks = invoked_record.most_common(index) if invoked_record else []
        return ranks

    @decorator_auth
    def render_GET(self, request):
        index = int(valid_index(request=request, arg="index", ins="int", default=10))
        ranks = self._invoke_rank(self.root.launcher.finished, index=index)  # 项目/爬虫列表
        return {"node_name": self.root.nodename, "status": "ok", "ranks": ranks}

```

**补充说明**：上一个 API 可以通过指定 index 的值来限制排行长度，而这个 API 在此基础上新增了排行的默认长度。将 index 的值设置为 0，这样的话假如排行中数量很多，在不指定 index 的情况下它将会使用默认值进行长度取值，有效控制了排行榜的长度。

增加路由映射：

```
invokerank.json = scrapyd.webservice.InvokeRank

```

使用测试：

![](https://user-gold-cdn.xitu.io/2018/10/15/1667704ae6114549?w=1713&h=443&f=gif&s=268689)

当不传入 index 时得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "ranks": [["fabias", 3], ["tips", 2]]}

```

当指定 index=1 时得到返回数据为：

```
{"node_name": "gannicus-PC", "status": "ok", "ranks": [["fabias", 3]]}

```

对比发现，默认排行长度此时为 3(小于默认值 10，不被截断)，但指定 index 后返回的结果中排行长度为 1，证明排行结果长度控制的测试也通过了。