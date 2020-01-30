# Scrapyd 源码目录

前面学习了 Scrapyd 的简介、文档及安装使用，基础部分已经学习完毕，或许你还有许多的疑问:

*   API 的功能是如何实现的？
*   Jobs 页面的爬虫运行状态是怎么统计出来的？
*   爬虫日志是如何生成的？
*   Scrapyd 配置文件在哪里？
*   Scrapyd 源码目录结构是什么样的？
*   ……

带着这些疑问，随我到源码中探个究竟。

## Scrapyd 的源码

[Scrapyd 的源码](https://github.com/scrapy/scrapyd)可以在 GitHub 上找到：

![](https://user-gold-cdn.xitu.io/2018/10/11/16661148609dd8ff?w=1374&h=954&f=gif&s=4190031)

它的文件目录如下图所示：

![](https://user-gold-cdn.xitu.io/2018/10/4/1663ef710d2c284a?w=975&h=632&f=png&s=97838)

从 GitHub 的代码提交记录来看，最近的代码更新是在 11个月以前，并且几乎每年都会有更新。代表这个项目是持续更新维护的，不用担心使用过后官方团队停止维护导致的项目管理隐患。

实际上我们安装 Scrapyd 之后，用到的代码是在上图`scrapyd`目录中，安装时也相当于将 Scrapyd 目录复制到 site-packages 内，至于其他的目录和文件，是在 PyPI 打包提交以及编写文档时所生成的文件。

## 启动文件、视图及 API

进入 Scrapyd 目录，下图为 Scrapyd 目录下的所有文件：

![](https://user-gold-cdn.xitu.io/2018/10/4/1663efc8353cee6b?w=981&h=833&f=png&s=132063)

这个小节我们**主要了解一下各个文件或目录的大致作用，知晓目录的结构以及文件分布**。

至于启动文件、视图、API 文件的源码调试与解析会在后面的小节中重点讲解，以下列出每个文件或目录的作用释义：

```
* scripts - 里面只有 1个可用的文件：scrapyd_run.py,
它是整个项目的启动文件。
* init.py - Scrapyd 启动前的配置。
* tests - 用于测试 Scrapyd 功能的代码集。
* VERSION - Scrapyd 版本号文件。
* app.py - Scrapyd 应用配置文件读取并进行设置。
* config.py - Scrapyd 配置及相关设置。
* default_scrapyd.conf -  Scrapyd 配置文件
* eggstorage.py - 项目打包设置及版本生成。
* eggutils.py - 项目打包。
* environ.py - 项目打包目录读取
* interfaces.py - 处理和存储项目包的检索以及爬虫队列。
* launcher.py - 执行爬虫运行、记录状态以及进程池相关。
* poller.py - 队列相关。
* runner.py - 任务具体执行。
* scheduler.py - 任务及项目的状态与记录更新。
* script.py - 用于执行 Scrapy 传递的命令。
* spiderqueue.py - 爬虫队列相关。
* sqllite.py - sqllite 数据库相关。
* txapp.py - 通过 Twisted 启动 Scrapyd。
* utils.py - 爬虫队列、项目列表 i 以及 JSON 视图。
* webservice.py - JSON 视图以及 Scrapyd 中提供的所有 API。
* website.py - Web 视图、Home、Jobs 等页面及功能。

```

## 启动文件

启动文件是整个项目的入口。用于启动项目的文件位于`/scrapyd/script`目录下，文件名为`scrapyd_run.py`，它通过载入指定`txapp.py`文件来启动`scrapyd`项目，`scrapyd_run.py`代码如下：

```
from twisted.scripts.twistd import run
from os.path import join, dirname
from sys import argv
import scrapyd

def main():
    argv[1:1] = ['-n', '-y', join(dirname(scrapyd.__file__), 'txapp.py')]
run()

```

[txapp.py](http://txapp.py) 则通过调用 get\_application 来获取 Scrapyd 配置并通过 RUN 命令启动 Scrapyd，负责初始化 Scrapyd 的 [txapp.py](http://txapp.py) 代码如下：

```
from scrapyd import get_application
application = get_application()

```

关于文件源码的深层次剖析与调试，会在后面的章节《[动手调试 Scrapyd 代码](https://juejin.im/book/5bb5d3fa6fb9a05d2a1d819a/section/5bbb471a5188255c5e66fef7)》，通过代码调试的方式理解 Scrapyd 的代码运行流程与逻辑。