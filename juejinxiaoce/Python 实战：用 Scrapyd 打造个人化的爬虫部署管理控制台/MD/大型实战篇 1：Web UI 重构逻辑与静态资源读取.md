# Web UI 重构逻辑与静态资源读取

通过前面的学习，我们已经了解了 Scrapyd 的目录结构、主要的代码分布以及通过调试不同类型的功能性代码，对 Scrapyd 中存放项目信息、爬虫信息的对象有了一定的理解。

然后我们动手编写了新的视图类`CustomResource`，编写了简单的 API，验证了对 Scrapyd 中的一些猜想。

现在我们对整个 Scrapyd 已经比较熟悉了，根据之前对 ScrapydArt 管理控制台的功能规划，我们可以绘制一个项目实践的思路图：

![](https://user-gold-cdn.xitu.io/2018/10/29/166c02482663b114?w=1100&h=770&f=png&s=39412)

*   首先读取静态文件；
*   然后根据规划编写首页的功能；
*   接着将首页功能与静态页结合，生成新的首页并对其进行测试；
*   既然首页有了新功能，那么 API 也要具备相应的功能；
*   API 作为整个项目的重点，应该具备更高级、更实用的功能；
*   然后对编写好的若干个 API 进行功能性测试；
*   最后对整个项目作一个总结；

**接下来我们就动手实践吧！**

## 界面重构说明

前面也提过，**Scrapyd 没有界面**，它的页面如下图演示：

![](https://user-gold-cdn.xitu.io/2018/10/15/16675684c0649ed9?w=1171&h=784&f=gif&s=316620)

虽然它是有部分 CSS 与 HTML，但是可以理解为没有界面（指的是样式）。既然我们已经熟悉了 Scrapyd，并且在其基础上增加功能，那就顺带连界面也改造吧。

> 笔者的前端水平还处于青铜段位

所以这次的前端界面我是从网上下载的，对其布局进行稍微改动，以满足 ScrapydArt 的展示需求。

![](https://user-gold-cdn.xitu.io/2018/10/15/166757d1968c052b?w=1857&h=927&f=gif&s=2682962)

为了方便大家使用，原版的 HTML 就不提供了，因为改动过程中非常容易出现错误，导致页面异常，影响学习进度。建议直接使用拆分好 header.html 与 footer.html，并且将变量填充好的 template，能够避免一些细小的错误，节省学习时间。

**template**下载地址：[点击下载](https://github.com/dequinns/ScrapydArt/tree/master/scrapydart/template)。

## 界面重构逻辑

回顾 Home 类中的界面部分的代码：

```
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
                ...html ...
        """ % vars
        return s.encode('utf-8')

```

其中的 `...html...`代表省略那部分代码，我们看主体代码逻辑即可。首先通过`self.root.scheduler.list_projects()`获取爬虫项目名列表，然后编写界面的 HTML 代码并将爬虫项目名列表以`format`的方式填充到 HTML 中，最后将 HTML 文本转成`bytes`格式并 return。

而回顾`Jobs`类中的界面部分的代码，实际上跟 Home 中是相同的，都是使用文本拼接的方式将 HTML 与项目数据结合并交给视图渲染呈现，只不过 Jobs 中多了对`css`的设置。

在 Web 框架中，后端与前端相关联时通常使用的是`MVC`模式，而 Scrapyd 中由于 HTML 部分比较少，所以它选择直接在后端代码中嵌入 HTML。由于 `Twisted.web` 的文档中 template 部分晦涩难懂且网上相关资料较少，所以笔者也不打算用标准的 MVC 模式来编写 ScrapydArt 的前端界面，而是采用与 Scrapyd 相同的办法，在后端代码中嵌入 HTML，但将所有的静态资源归类放到`template`目录下，模拟 `MVC`的模式。

## 界面重构 - 静态资源

通过上方的连接地址下载静态模板，放到 Scrapyd 目录内，也就是与配置文件同级。既然是要模拟 MVC，那么就需要一个方法来读取所有的静态资源，在 Scrapyd 目录内新建名为`websource.py`的文件：

```
import os
# custom import in here

HEADER_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/header.html"
FOOTERS_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/footers.html"
INDEX_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/index.html"
JOBS_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/jobs.html"
FEATURE_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/feature.html"
DOCUMENTS_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/documents.html"
STYLE_CSS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/css/style.css"
RESET_CSS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/css/reset.css"
JQUERY_JS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/js/jquery-2.1.4.js"
MAIN_JS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/js/main.js"
MODERN_JS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/js/modern.js"
VELOCITY_MIN_JS = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/js/velocity.min.js"
SVG = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/img/cd-icon-arrow.svg"

```

以上的代码是通过 Python 的`os`获取所有使用到的静态资源路径，再读取其中的文本内容，因为读取是一个可复用的操作，所以这里单独写一个函数来读取指定文件的文本，在`websource.py` 同级目录新建`webtools.py`文件：

```
def file_read(filename):
    # 打开指定文件读取文本内容并转换成 str 格式并返回
    with open(filename, 'r', encoding="utf-8") as h:
        text_bytes = h.read().encode("utf-8")
        text_str = str(text_bytes, encoding="utf-8")
    return text_str

```

定义好`file_read`函数后，回到`websource.py`文件中将刚才的代码改成(其中的省略号代表文章中省略，但是**实际代码中需要保持与上方代码一致**)：

```
import os
# custom import in here
from .webtools import file_read


HEADER_HTML = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/header.html"
...
 ...
SVG = os.path.dirname(os.path.dirname(__file__)) + "/scrapydart/template/img/cd-icon-arrow.svg"


FILES = [HEADER_HTML, FOOTERS_HTML, INDEX_HTML, JOBS_HTML, FEATURE_HTML,
         DOCUMENTS_HTML, STYLE_CSS, RESET_CSS, JQUERY_JS, MAIN_JS,
         MODERN_JS, VELOCITY_MIN_JS, SVG]


files = list(map(file_read, FILES))

```

完成静态资源的读取后，接下来就要到`website.py`文件中对 Home 类进行改造。