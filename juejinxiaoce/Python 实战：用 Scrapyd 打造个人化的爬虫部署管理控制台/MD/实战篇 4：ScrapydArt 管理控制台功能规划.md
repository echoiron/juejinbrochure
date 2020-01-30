# ScrapydArt 管理控制台功能规划

作为一名职业的爬虫工程师，在日常的爬虫维护管理过程中需要用到非常多的工具以及多方面的数据信息以更好、更轻松的完成工作。

> 举个例子

假设在 Scrapyd 上部署了 8个项目，每个项目中又包含不同数量的爬虫，你如何能够知道：

*   哪一个爬虫被调用的次数是最多的？
*   运行时长排行前 20 的是哪些爬虫？
*   如果向对爬虫运行记录进行排序呢？
*   怎么筛选指定范围内的爬虫运行记录？
*   ... ...

Scrapyd 目前的功能无法满足工作需求，作为一名动手能力强的程序员，我们可以在其基础上增加新功能、重构页面，让爬虫的管理工作更轻松、爬虫运行状况的观察更直观。所以，我们需要对新的 Scrapyd 的功能进行规划，在保持风格一致的情况下尽可能多的贴近日常工作需求。

## 功能规划

既然要对新的 Scrapyd 的功能进行规划，那我们就应该将已有功能与具体的需求功能列出，再完善每一个具体的需求功能的作用以及功能模块结构图。

Scrapyd 已有功能中，仅列出日常工作所使用到的类及其功能:

```
1、WebUI：
Home 类
    显示当前已部署的项目名称。
    提供 jobs、logs 的跳转链接。
    给出 cURL 使用示例。
    提供官方文档的跳转链接。
Jobs 类
    展示爬虫的运行结果信息，包括启动项目名、爬虫名、启动时间、运行时长、结束时间以及运行日志等。
    记录爬虫当前时刻不同状态的数量，如挂起状态、正在运行、运行结束。

2、API（官方提供了 10个 API，用于查看项目信息或爬虫信息）：
API 及对应功能描述
    daemonstatus.json 用以检查 Scrapyd 服务的爬虫状态及对应数量。
    addversion.json 用以为项目添加版本，如果项目不存在则创建项目。
    schedule.json-用以启动指定的爬虫。
    cancel.json-用以取消指定的爬虫。
    listprojects.json-用以查看当前已部署的项目名称。
    listversions.json-用以查看指定项目的版本号。
    listspiders.json-用以查看指定项目的爬虫名称。
    listjobs.json-用以查看指定项目下爬虫的运行状态。
    delversion.json-用以删除指定项目的指定版本，如版本不存在则将项目删除。
    delproject.json-用以删除指定项目及其已上传的所有版本。
API 父类
    JsonResource
    resource.Resource

3、Scrapyd 路由及配置
Root 类
    路由配置。
    更新项目信息。
    读取配置文件并作相关设置
配置文件
    default_scrapyd.conf

```

**Scrapyd 新功能**

部分新功能将在原基础上进行增强，而另一部分则是根据需求进行设计。以下功能列表中 `-`代表删减该项目，`+`代表新增该项目，`-->`代表改变该项目，`*`代表无序列表，`x`代表在规划中但有可能不予实现的项目。

```
1、WebUI：
Home 类
    提供 jobs、logs 的跳转链接。
    给出 cURL 使用示例。
    - 提供官方文档的跳转链接。
    - 显示当前已部署的项目名称列表。
    + 重构界面
    + 当前已部署的项目名称及对应爬虫名称和爬虫数量，以表格形式展示。
    + 爬虫调用情况统计信息，包括：被调用的爬虫名称、未被调用的爬虫名称、被调用次数最多的爬虫及其对应次数。
    + 爬虫运行与统计信息，包括：
        * 爬虫当前时刻不同状态的数量，如挂起状态、正在运行、运行结束
        * 爬虫运行时长统计，如爬虫平均运行时长、最长时长、最短时长。
        * 当前项目总数
        * 当前爬虫总数
    + rank榜单
        * 爬虫运行时长排行榜
        x 爬虫被调用次数排行榜
Jobs 类
    展示爬虫的运行结果信息，包括启动项目名、爬虫名、启动时间、运行时长、结束时间以及运行日志等。
    记录爬虫当前时刻不同状态的数量，如挂起状态、正在运行、运行结束。
    + 重构界面
    + 给出 cURL 使用示例。
    + 爬虫调用情况统计信息，包括：被调用的爬虫名称、未被调用的爬虫名称、被调用次数最多的爬虫及其对应次数。

2、API（官方提供了 10个 API，用于查看项目信息或爬虫信息）：
API 父类
    JsonResource
    resource.Resource
    + CustomResource
API 及对应功能描述
    daemonstatus.json 用以检查 Scrapyd 服务的爬虫状态及对应数量。
    addversion.json 用以为项目添加版本，如果项目不存在则创建项目。
    schedule.json-用以启动指定的爬虫。
    cancel.json-用以取消指定的爬虫。
    listprojects.json-用以查看当前已部署的项目名称。
    listversions.json-用以查看指定项目的版本号。
    listspiders.json-用以查看指定项目的爬虫名称。
    listjobs.json-用以查看指定项目下爬虫的运行状态。
    delversion.json-用以删除指定项目的指定版本，如版本不存在则将项目删除。
    delproject.json-用以删除指定项目及其已上传的所有版本。
    + schedulelist.json-用以获取爬虫调用情况。
    + runtimestats.json-用以获取爬虫运行时间统计。
    + psnstats.json-用以获取爬虫项目及其数量统计。
    + prospider.json-用以获取项目与对应爬虫名以及数量统计。
    + timerank.json-用以获取爬虫运行时长榜。
    + invokerank.json-用以获取爬虫调用排行榜。
    + filter.json-根据参数按时间范围/项目名称/爬虫名称/运行时长对爬虫运行记录进行筛选过滤。
    + order.json-根据参数按时间范围/项目名称/爬虫名称/运行时长对爬虫运行记录进行排序。

3、Scrapyd 路由及配置
Root 类
    路由配置。
    更新项目信息。
    读取配置文件并作相关设置
配置文件
    default_scrapyd.conf --> 改变配置级别、新增配置参数
    
+ 4、访问权限控制
    + 权限控制装饰器

```

### 新的名字

对于新的平台，总得起一个合乎情理的名字。根据规划的功能来看，主要在这几个方面：`auth`、`resource`、`template`。

前面的名字沿用 Scrapyd，将新增的这几个方面的英文缩写放到一起，组合起来即是 art，那么我就将它称为`ScrapydArt`。

### 效果预览图

> Web 界面及功能演示

![](https://user-gold-cdn.xitu.io/2018/10/12/166655ce18edbde0?w=800&h=428&f=gif&s=1014356)

> 新增 API 及数据结果演示

比如根据指定的`project`名称筛选出对应的爬虫记录。

![](https://user-gold-cdn.xitu.io/2018/10/16/1667aca84676c4ad?w=1022&h=764&f=gif&s=272771)

> 访问权限控制演示

先测试未携带用户名和密码参数的请求：

![](https://user-gold-cdn.xitu.io/2018/10/15/1667684b9c245f37?w=1723&h=828&f=gif&s=135927)

得到的是未取得授权的提示

```
{"status": "error", "message": "You have not obtained the authorization."}

```

再试一下携带用户名和密码参数，但值与配置文件不符：

![](https://user-gold-cdn.xitu.io/2018/10/15/1667686c881256a8?w=1730&h=823&f=gif&s=57643)

得到的是用户名或密码不正确的提示：

```
{"status": "error", "message": "username or password you entered is incorrect. Please re request"}

```

再试试正确的用户名和密码：

![](https://user-gold-cdn.xitu.io/2018/10/15/166768a8db3253c7?w=1711&h=808&f=gif&s=478998)