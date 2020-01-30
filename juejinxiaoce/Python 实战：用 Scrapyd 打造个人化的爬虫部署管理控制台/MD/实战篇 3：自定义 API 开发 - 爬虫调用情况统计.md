# 自定义 API 开发 - 爬虫调用情况统计

前面动手编写了`CustomResource`，并且将其应用到原来的 HTML 视图类上。这一节我们来学习如何编写 API。

在之前对 API 的阅读中，我们知道 API 代码写在`scrapyd/webservice.py`中，为了保持风格一致，我们也将代码写在此文件中。

## 编写自定义 API

在之前的 API 源码与视图类源码的阅读中，我们知道渲染由 render 方法完成，并且可以通过`render_GET`以及`render_POST`来指定请求方式。

### 编写 GET 类型的爬虫调用信息统计 API

*   功能：爬虫调用情况
*   描述：如被调用过的爬虫、未被调用过的爬虫、被调用次数最多的爬虫名称及次数

在`webservice.py`中新建类`ScheduleList`，它继承之前我们自己编写的视图类`CustomResource`：

```
class ScheduleList(CustomResource):

```

通过对其他 API 的调试，可以知道所有运行完毕的爬虫运行信息对象以列表的形式记录在`self.root.launcher.finished`中：

```
[
<scrapyd.launcher.ScrapyProcessProtocol object at 0x10cf97d68>,
<scrapyd.launcher.ScrapyProcessProtocol object at 0x10cf546d8>,
<scrapyd.launcher.ScrapyProcessProtocol object at 0x10cf3d4e0>
]

```

![](https://user-gold-cdn.xitu.io/2018/10/11/16662fee3bc4f760?w=1536&h=1002&f=gif&s=4078380)

![finished object](https://user-gold-cdn.xitu.io/2018/10/5/16643ab60543f5ff?w=686&h=264&f=png&s=69929)

每个对象中包含对应爬虫相关运行信息，如项目名称 project、爬虫名称 spider、启动时间 start\_time、结束时间 end\_time、jobid、日志路径 logfile 等。

根据需求，要取的数据有：

```
* 所有爬虫名称
* 已运行完毕的爬虫名称
* 爬虫调用记录

```

通过 A 与 B 的差值计算得出未被调用过的爬虫、通过调用记录计算爬虫调用次数并取最大次数。

代码逻辑：

```
* 取得运行完毕的爬虫运行记录列表
* 获取爬虫列表
* 计算差值，得出已被调用与未被调用的爬虫名列表
* 计算爬虫调用记录中被调用次数最多的爬虫名及其次数
* 将结果以原 JSON 风格渲染输出

```

相应代码：

```
class ScheduleList(CustomResource):
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

其中通过`get_ps`方法获取爬虫列表：

```
def get_ps(self):
    """获取项目列表与爬虫列表
    :param self:
    :return: projects-项目列表， spiders-爬虫列表
    """
    projects = list(self.root.scheduler.list_projects())
    spiders = get_spiders(projects)
    return projects, spiders

```

通过`get_invokes`方法计算所需结果：

```
def get_invokes(finishes, spiders):
        """获取已被调用与未被调用的爬虫名称及被调用次数最多的爬虫
        :param: finishes 已运行完毕的爬虫运行信息记录列表
        :param: spiders 所有爬虫名列表
        :return: invoked-被调用过的爬虫集合, un_invoked-未被调用的爬虫集合, most_record-被调用次数最多的爬虫名与次数
        """
        invoked = set(i.spider for i in finishes)
        un_invoked = set(spiders).difference(invoked)
        invoked_record = Counter(i.spider for i in finishes)
        most_record = invoked_record.most_common(1)[0] if invoked_record else ("nothing", 0)
        return invoked, un_invoked, most_record

```

还需要通过:

```
def get_spiders(values):
    if not values:
        return []
    # [['tips', 'nof'], ['nop']] -> ['tips', 'nof', 'nop']
    value = list(reduce(lambda x, y: x+y,  map(get_spider_list, values)))  # first 2.8s
    return value

```

## 自定义 API 的使用

### 配置路由

代码编写好之后，还需要在为其配置路由映射规则。打开 Scrapyd 的配置文件`default_scrapyd.conf`，在`services`级下添加路由，如：

```
[services]
schedulelist.json = scrapyd.webservice.ScheduleList

```

![](https://user-gold-cdn.xitu.io/2018/10/11/166630dc666d4c87?w=1074&h=424&f=gif&s=1562870)

保存之后重新启动 Scrapyd 服务才能生效。

### API 的使用

由于编写代码时选择 `render_GET` 方法，所以在使用时与原 API 使用方式一致，如果是使用 cURL 模拟请求，则请求语句为:

```
curl: http://localhost:6800/schedulelist.json

```

如果是使用 Postman 作为模拟请求工具，则：

![](https://user-gold-cdn.xitu.io/2018/10/11/1666310ea7f2c5d0?w=1078&h=876&f=gif&s=786048)

如果使用代码（requests）发起请求，代码为：

```
import requests

resp = requests.get("http://localhost:6800/schedulelist.json")
print(resp.text)

```

发起请求后，得到的返回结果为：

```
{
"node_name": "node-name",
"status": "ok",
"invoked_spider": ["quinns","fabias"],
"un_invoked_spider": ["artspider"],
"most_record": ["fabias",3]
    
}

```

返回结果中键的含义解释：

```
* node_name-请求发起者
* status-请求状态，ok 为成功
* invoked_spider-被调用过的爬虫列表
* un_invoked_spider-未被调用过的爬虫列表
* most_record-被调用次数最多的爬虫名及次数

```

## 小结

基于之前几个小节学习的知识，结合需求分析，render 选型和逻辑分析，逐步完成了自定义 API 的编写。