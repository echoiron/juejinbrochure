# 动手调试 Scrapyd 代码

通过前面几个小节的学习，我们了解了 Scrapyd 的基础知识、Scrapyd 的使用、 [Scrapyd 源码目录](https://juejin.im/book/5bb5d3fa6fb9a05d2a1d819a/section/5bb5ed2ff265da0aa06f08ac)及[视图类](https://juejin.im/book/5bb5d3fa6fb9a05d2a1d819a/section/5bb7206f5188255c3f6beeb7)，逐步深入的领略了 Scrapyd 整个结构和编码风格。但是：

```
1、如何才能够更深入的了解它的运行流程呢？
2、爬虫的运行记录会存放在哪里呢？
3、在原有的 Scrapyd 代码中，有哪些可以为我所用？
4、它的 API 由哪些元素构成？
5、我们可以改动现有函数或方法吗？
6、... ...

```

对于以上列出的问题，我们不得而知，但又不得不知。

```
* 不得而知：因为之前的小节内容中，并未深入讲解 Scrapyd 的源码，仅停留在使用与结构上；
* 不得不知：对于一个框架的学习，必须深入到代码中，通过对代码的调试才能够在脑海中绘制出框架的图谱；

```

所以，在这里我将会用代码调试的方式，跟大家一起深入的了解 Scrapyd，看一看爬虫运行记录存放位置，筛选可以为我所用的 Scrapyd 代码。拆解它的 API，看一看 API 由哪些元素构成，试一试能不能改动现有函数。

只有这样，我们才能够在脑海中绘制出清晰的 Scrayd 图谱，为我们自己能够编写出与 Scrapyd 风格一致的代码夯实基础。

> 为什么要风格一致？

在框架代码的改动过程中，应尽量与原框架代码保持一致的风格，有助于后期代码管理和维护，同时也能够减少一些不易察觉的错误。

## Scrapyd 的代码调试

在调试之前，我们想一想：选择哪一部分的代码进行调试会比较好呢？

根据之前小节对 Scrapyd 源码目录结构以及常用功能的学习，我认为选择启动文件、首页以及 API 中的`listspiders.json`进行调试，能够让我们在短时间内对 Scrapyd 有更深一层的理解。

### 启动文件代码调试

#### scrapyd\_run.py

启动文件位于`scrapyd/scripts`目录下，名为`scrapyd_run.py`，文件中代码不多：

```
#!/usr/bin/env python

from twisted.scripts.twistd import run
from os.path import join, dirname
from sys import argv
import scrapyd


def main():
    argv[1:1] = ['-n', '-y', join(dirname(scrapyd.__file__), 'txapp.py')]
    run()

if __name__ == '__main__':
    main()

```

#### Import 代码释义

Python 的代码自上而下执行，那就让我们从 Imoprt 开始：

```
from twisted.scripts.twistd import run   
from os.path import join, dirname
from sys import argv
import scrapyd

```

导入这些包的作用是什么呢？

```
# 先从 Scrapyd 的底层框架 Twisted 中导入用于启动 Twisted 的 Run 方法
# 然后从 os.path 中导入用于操作文件路径的 join 和 dirname 方法
# 再从 sys 中导入获取命令行参数的 argv 方法
# 最后导入 Scrapyd 项目本身

```

#### main 函数与 argv

接着定义了`main`函数：

```
def main():
    argv[1:1] = ['-n', '-y', join(dirname(scrapyd.__file__), 'txapp.py')]
    run()

```

函数中的 argv 与其他参数到底有什么作用呢？我们可以通过 PyCharm 在 run() 打断点，让其运行到 run() 时暂停，这样我们就可以观察 argv 了，如下图所示：

![](https://user-gold-cdn.xitu.io/2018/10/11/16662d856b8acc24?w=1538&h=998&f=gif&s=2808159)

Debug 后得到 argv 的值为：

```
 ['/Volumes/HFSFILE/github/scrapyd/scrapyd/scripts/scrapyd_run.py', '-n', '-y', '/Volumes/HFSFILE/github/scrapyd/scrapyd/txapp.py']

```

它是一个列表，列表的第一个元素是 `scrapyd_run.py` 的文件路径，也就是`argv`代码所获得的路径值，argv 会默认将当前文件路径添加到列表的第一个元素；列表最后一个元素是文件 `txapp.py` 的路径，也就是`join(dirname(scrapyd.__file__), 'txapp.py')`这部分代码获取到的值。那么，`-n`和`-y`到底是什么呢？

为了更好的观察，这里将 argv 这一整行代码注释掉：

```
def main():
    # argv[1:1] = ['-n', '-y', join(dirname(scrapyd.__file__), 'txapp.py')]
    run()

```

然后直接运行`scrapyd_run.py`文件。

发现运行失败，并在 PyCharm 的运行记录框中给出了提示，如下图：

![](https://user-gold-cdn.xitu.io/2018/10/11/16662dba20cf559c?w=1540&h=1002&f=gif&s=3478686)

提示中出现了`-n`和`-y`参数，它们的含义是什么？

```
-n, --nodaemon don't daemonize, don't use default umask of 0077
-y, --python=  read an application from within a Python file (implies -o)

```

*   \-n 代表不以守护进程模式启动
*   \-y 代表从 Python 文件中读取应用程序

回头再来看看 argv 的作用，[Python 文档中对 argv 的介绍](https://docs.python.org/3.6/library/sys.html?highlight=argv#sys.argv)如下：

> sys.argv
> 
> `The list of command line arguments passed to a Python script. argv[0] is the script name` (it is operating system dependent whether this is a full pathname or not). If the command was executed using the -c command line option to the interpreter, argv\[0\] is set to the string '-c'. If no script name was passed to the Python interpreter, argv\[0\] is the empty string.

其中我对释义做了高亮，那段释义翻译成中文：

> 传递给 Python script 的命令行参数列表，argv\[0\] 是 script 文件名

这也解释了，为什么 argv\[1:1\] 列表的第一个元素是 scrapyd\_run.py 的路径，其余的释义并不重要，在这里可以将其略过。

#### txapp

那么，传递进来的 txapp.py 文件，又是做什么的呢？这是它的代码：

```
# this file is used to start scrapyd with twistd -y
from scrapyd import get_application
application = get_application()

```

它从 Scrapyd 项目中导入`get_application`方法，通过`Ctrl/Commond+鼠标左键` 跟进`get_application`方法的代码：

```
from scrapy.utils.misc import load_object
from scrapyd.config import Config


def get_application(config=None):
    if config is None:
        config = Config()
    apppath = config.get('application', 'scrapyd.app.application')
    appfunc = load_object(apppath)
    return appfunc(config)

```

它的逻辑很简单：

*   判断是否传入指定的 config。 如果有则使用指定的 config，如无则使用框架预先定义的`Config`
*   初始化`Config`并使用`get()与_getany()`，最终获得传入的字符串`'scrapyd.app.application'`
*   使用 `appfunc = load_object(apppath)`获得 mod 对象的 application 属性对象
*   最后使用 `appfunc(config)` 将得到的`Componentized`对象返回给`txapp`中的`application` 变量

> 至此，整个 Scrapyd 服务的启动就完成了。

或许你还有疑问，为什么要返回`Componentized`对象给`application`变量呢?`run()`方法执行了什么呢？

这里我们学习的是 Scrapyd，能够大致理解整个流程即可，对于十分细致的 Twisted 方法及函数还需翻阅 Twisted 文档。通过上面的代码跟进及调试，我们基本可以理清它们的顺序和关系：

*   在启动之前，通过`scrapyd_run.py`的`argv`设置了基本参数：`-n`和`-y`参数。并且将`txapp`的`application`对象以参数的形式与`-n`和`-y`一并传递给了`twisted`(application 是固定的参数名)。
*   `run()`方法是以`argv`传递对应的参数作为启动配置的，`run()`执行时，就启动了整个项目。

所以，整个 Scrapyd 服务由`run()`方法唤起，但是唤起前将`txapp.py`所获得的`Componentized`对象以及`-n`和`-y`参数传递给`twisted`。

## 首页代码调试

首页相关的代码在`website.py`中的`Home`类中,其代码如下：

```

class Home(resource.Resource):

    def __init__(self, root, local_items):
        resource.Resource.__init__(self)
        self.root = root
        self.local_items = local_items

    def render_GET(self, txrequest):
        vars = {
            'projects': ', '.join(self.root.scheduler.list_projects())
        }
        s = """
            <html>
            <head><title>Scrapyd</title></head>
            <body>
            <h1>Scrapyd</h1>
            <p>Available projects: <b>%(projects)s</b></p>
            <ul>
            <li><a href="/jobs">Jobs</a></li>
            """ % vars
                    if self.local_items:
                        s += '<li><a href="/items/">Items</a></li>'
                    s += """
            <li><a href="/logs/">Logs</a></li>
            <li><a href="http://scrapyd.readthedocs.org/en/latest/">Documentation</a></li>
            </ul>
            
            <h2>How to schedule a spider?</h2>
            
            <p>To schedule a spider you need to use the API (this web UI is only for
            monitoring)</p>
            
            <p>Example using <a href="http://curl.haxx.se/">curl</a>:</p>
            <p><code>curl http://localhost:6800/schedule.json -d project=default -d spider=somespider</code></p>
            
            <p>For more information about the API, see the <a href="http://scrapyd.readthedocs.org/en/latest/">Scrapyd documentation</a></p>
            </body>
            </html>
        """ % vars
        return s.encode('utf-8')

```

在 Web 页面中界面如下图所示：

![web index](https://user-gold-cdn.xitu.io/2018/10/9/16657440a52d28a5?w=805&h=341&f=png&s=29993)

### render\_GET

通过 Web 页面与 Home 类中的代码对比，不难发现数据的渲染都是在`render_GET`方法中，`render_GET`由以下三个部分组成：

*   vars - 爬虫项目列表
*   s - 嵌入爬虫项目列表 vars 的 html 文本
*   return - 将文本数据编码后返回给 render 方法，最后呈现到页面

那`render_GET`与`render`有什么关系？为什么它通过`return`就能够将页面呈现出来？这部分在 [Scrapyd 视图类](https://juejin.im/book/5bb5d3fa6fb9a05d2a1d819a/section/5bb7206f5188255c3f6beeb7)已经做了讲解，这里就不再多提。

### 爬虫项目的数据来源

上面提到，`vars`负责将爬虫项目列表取出，这部分的代码是：

```
vars = {'projects': ', '.join(self.root.scheduler.list_projects())}

```

真正负责取数据的是`self.root.scheduler.list_projects()`,我们将代码拆成`self.root.scheduler`与`self.root.scheduler.list_projects()`并对其进行调试，看看这段代码包含了哪些数据信息，先将 Home 中 `render_GET` 的代码改成（省略号代表下面的代码保持不变，此处省略）：

```
    def render_GET(self, txrequest):
        scheduler = self.root.scheduler
        pro = scheduler.list_projects()
        vars = {
            'projects': ', '.join(self.root.scheduler.list_projects())
        }
        ... ...

```

新增加了`scheduler`和`pro`，并在变量 `vars`处打断点，然后通过 PyCharm 的`debug`按钮运行`scrapyd_run.py`文件。待 Scrapyd 服务运行起来后，在浏览器访问`localhost:6800`，程序运行后就会暂停在断点 vars 处，如下图所示：

![](https://user-gold-cdn.xitu.io/2018/10/9/166575b9887eb330?w=1214&h=720&f=png&s=234005)

### Encode

在最后`return`的时候，需要对`html`文本进行编码转化,将原本的`str`转成`bytes`格式数据，也就是：

```
return s.encode('utf-8')

```

如果直接将 `str` 返回的话，会得到`Request did not return bytes`的错误提示。

## API 代码调试

API 作为 Scrapyd 使用最广泛并且最灵活的部分，是 Scrapyd 的功能担当。假如没有这些 API，对于 Scrapyd 的使用者来说，在爬虫项目管理方面将变得困难。

### listspiders.json

Scrapyd 所有 API 对应的类都在 `webservice.py` 文件中，本次将 `listspiders.json` 作为调试的案例，其代码为：

```
class ListSpiders(WsResource):
    def render_GET(self, txrequest):
        args = native_stringify_dict(copy(txrequest.args), keys_only=False)
        project = args['project'][0]
        version = args.get('_version', [''])[0]
        spiders = get_spider_list(project, runner=self.root.runner, version=version)
        return {"node_name": self.root.nodename, "status": "ok", "spiders": spiders}

```

我们来大致分析一下，这个类自上而下做了什么：

*   首先，这个类继承自 WsResource
*   其次，它同样是使用了 `render_GET` 来渲染数据
*   通过 `native_stringify_dict` 方法获取请求时携带的参数
*   根据传递的爬虫项目名称获取 project 对象以及版本 Version
*   通过 `get_spider_list` 方法以及对应参数获取爬虫名称列表
*   最后将爬虫列表结果以 json 格式返回

从逻辑上可以看到，关键的地方就在于`args = native_stringify_dict(copy(txrequest.args), keys_only=False)`，需要在使用时传递指定的爬虫项目名称。

如果不传递任何参数，则返回错误提示: `{"node_name": "GannicusdeiMac", "status": "error", "message": "'project'"}`

如果传递了错误的参数名或者不存在的项目名称，同样会得到错误提示。所以在 [Scrapyd 文档中 API 示例部分](https://scrapyd.readthedocs.io/en/stable/api.html#listspiders-json)强调了必须传递参数：

![doc](https://user-gold-cdn.xitu.io/2018/10/10/1665e3b2cdcdf5b2?w=791&h=221&f=png&s=21569)

那传递的参数是以什么形式出现在代码中的呢？我们可以对其进行调试，在`args` 的下一行（也就是 project）处打断点，并且通过 PyCharm 的 Debug 按钮启动项目，然后在浏览器访问`http://localhost:6800?project=arts`,待代码运行到断点处就可以看到 args 所获的值为`{'project': ['arts']}`，是以字典的形式出现在代码中。

所以下一句代码取参数中的 project 键中第 0个位置的值，以刚才的请求作为例子，就是取到`arts`这个名字。

再往下，通过 `get_spider_list` 方法以及传递的项目名称（arts）获取爬虫的名称列表。

当请求参数是正确的，并且项目名称也正确，就会得到带有爬虫名称的 json 数据：

`{"node_name": "GannicusdeiMac", "status": "ok", "spiders": ["artspider"]}`

那在 `get_spider_list` 方法中，又完成了哪些事呢？让我们通过 `Ctrl/Commond+鼠标左键` 跟进代码：

```
def get_spider_list(project, runner=None, pythonpath=None, version=''):
    """Return the spider list from the given project, using the given runner"""
    if "cache" not in get_spider_list.__dict__:
        get_spider_list.cache = UtilsCache()
    try:
        return get_spider_list.cache[project][version]
    except KeyError:
        pass
    if runner is None:
        runner = Config().get('runner')
    env = os.environ.copy()
    env['PYTHONIOENCODING'] = 'UTF-8'
    env['SCRAPY_PROJECT'] = project
    if pythonpath:
        env['PYTHONPATH'] = pythonpath
    if version:
        env['SCRAPY_EGG_VERSION'] = version
    pargs = [sys.executable, '-m', runner, 'list']
    proc = Popen(pargs, stdout=PIPE, stderr=PIPE, env=env)
    out, err = proc.communicate()
    if proc.returncode:
        msg = err or out or ''
        msg = msg.decode('utf8')
        raise RuntimeError(msg.encode('unicode_escape') if six.PY2 else msg)
    # FIXME: can we reliably decode as UTF-8?
    # scrapy list does `print(list)`
    tmp = out.decode('utf-8').splitlines();
    try:
        project_cache = get_spider_list.cache[project]
        project_cache[version] = tmp
    except KeyError:
        project_cache = {version: tmp}
    get_spider_list.cache[project] = project_cache
    return tmp

```

**代码比较长，我们挑重点来理解：**

*   首先，看方法的注释说明，大意是根据指定的爬虫名返回爬虫列表。
*   优先从爬虫缓存列表中读取记录，如果有则直接返回结果。
*   如无则尝试从数据库中读取。
*   最后更新项目缓存并将结果返回。

官方注释原文为： `Return the spider list from the given project, using the given runner`

优先从爬虫缓存列表中读取记录，如果有则直接返回结果：

```
return get_spider_list.cache[project][version]

```

尝试从数据库中读取代码为：

```
env['SCRAPY_PROJECT'] = project
pargs = [sys.executable, '-m', runner, 'list']
proc = Popen(pargs, stdout=PIPE, stderr=PIPE, env=env)
out, err = proc.communicate()

```

更新项目缓存并将结果返回代码为：

```
tmp = out.decode('utf-8').splitlines();
    try:
        project_cache = get_spider_list.cache[project]
        project_cache[version] = tmp
    except KeyError:
        project_cache = {version: tmp}
    get_spider_list.cache[project] = project_cache
    return tmp

```

## 关于爬虫名称列表的获取速度

用`get_spider_list`方法获取爬虫名称的速度并不快，甚至可以说有点慢，所以它才会使用缓存来提升读取速度。方法中`out, err = proc.communicate()`这句代码的运行时长约为 1.3-2.5秒，笔者通过不断的实验发现，它里面调用的其他方法中使用了`while`以及`for 循环`，每个循环运行的时间并不长，但是多个循环的时间累加起来就变成了1.3-2.5秒。

## 小结

本小节我们从需求与问题开始，再到代码调试的原因以及选择需要调试的几种不同类型的代码进行深入观察。在整个过程中了解到了 Scrapyd 的代码结构、代码运行流程、代码风格以及爬虫运行记录的存放处。**由此得到一个启发：**

> 对于后面的功能模块开发，我们可以从现有的代码中拿到部分资源，通过对资源的类型转换或者再次计算，得到我们需要的数据结果。

比如可以通过观察`self.root`获取爬虫运行的相关信息、使用`get_spider_list`方法可以获得指定项目中的爬虫名列表、使用`native_stringify_dict`方法可以获得传递的参数键值对等等。