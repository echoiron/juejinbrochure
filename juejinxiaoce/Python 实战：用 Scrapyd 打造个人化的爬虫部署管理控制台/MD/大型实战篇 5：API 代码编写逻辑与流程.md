# API 代码编写逻辑与流程

既然 Web 界面中新增了如此多且重要的功能，那么 API 也不能落后啊。并且 API 作为整个 Scrapyd 或者说 ScrapydArt 的规划重点是有一定的道理的，JSON 格式的数据无论传输或者后续扩展方面都比 HTML 的灵活，并且在使用上也是非常的方便。

## 新增 API 说明

根据之前对 ScrapydArt 平台的功能规划：

```
    + schedulelist.json-用以获取爬虫调用情况。
    + runtimestats.json-用以获取爬虫运行时间统计。
    + psnstats.json-用以获取爬虫项目及其数量统计。
    + prospider.json-用以获取项目与对应爬虫名以及数量统计。
    + timerank.json-用以获取爬虫运行时长榜。
    + invokerank.json-用以获取爬虫调用排行榜。
    + filter.json-根据参数按时间范围/项目名称/爬虫名称/运行时长对爬虫运行记录进行筛选过滤。
    + order.json-根据参数按时间范围/项目名称/爬虫名称/运行时长对爬虫运行记录进行排序。

```

可以看出，一部分功能代码已经在 Web 重构时编写好了，剩下 filter.json、order.json 的功能需要从头开始编写。

## 实现流程

之前已经尝试编写过一个功能简单的 API，对 API 的编写已经有了具体的实践与经验，但这里还是回顾一下编写一个 API 的流程：

![](https://user-gold-cdn.xitu.io/2018/10/15/16676d99db03a1ca?w=1644&h=147&f=png&s=22990)

通过上面的流程图可以发现，整个 API 的编写过程很简单，只需要将具体的功能性代码编写好，然后使用 JSON 格式将数据结果返回，最后到配置文件中添加 API 的路由映射配置即可。