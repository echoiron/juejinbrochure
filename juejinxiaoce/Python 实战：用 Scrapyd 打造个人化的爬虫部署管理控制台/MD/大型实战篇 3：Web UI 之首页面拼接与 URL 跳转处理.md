# Web UI 之首页页面拼接与 URL 跳转处理

编写完首页的功能之后，要将页面与数据结合起来，拼接成完整的`html`并渲染呈现。

## 界面改造-首页页面拼接

将上面定义的静态资源方法`files`与数据计算方法`features`导入`website.py`：

```
from .websource import files
from .webtools import features

```

将 website.py 中的 microsec\_trunc 方法放到 webtools.py 中，并且在 website.py 中将其导入：

```
from .webtools import  microsec_trunc

```

接着将静态资源拆包，考虑到 Jobs 当中还需要用到静态资源，所以拆包操作放在 Root 类之前：

```
HEADER_HTML, FOOTERS_HTML, INDEX_HTML, JOBS_HTML, FEATURE_HTML, \
DOCUMENTS_HTML, STYLE_CSS, RESET_CSS, JQUERY_JS, MAIN_JS, \
MODERN_JS, VELOCITY_MIN_JS, SVG = files

# class Root(resource.Resource):
# ... ...

```

然后到 Home 类的 render\_GET 中将数据计算结果拆包：

```
    def render_GET(self, request):
        pending, running, finished, average, shortest, \
        longest, projects, spiders, ranks, dps, \
        lpn, lsn, invoked_spider, un_invoked_spider, most_record = features(self)

```

因为 HTML 中表格部分`<tr><td></td></tr>`结构用 Python 代码根据计算结果动态生成

![](https://user-gold-cdn.xitu.io/2018/10/15/1667670018dc5d86?w=1440&h=442&f=png&s=50418)

所以 HTML 表格部分代码中只保留`<table><tbody>...</tbody></table>`结构。这里到 `webtools.py` 中定义一个动态生成`<tr><td></td></tr>`结构的函数`make_table`:

```
def make_table(values):
    tds = []
    for v in values:
        project_name, spider_name, number = v.get("project"), v.get("spider"), v.get("number")
        aps = "<tr><td><i>{project_name}</i></td><td>{spider_name}</td><td>{spider_num}</td></tr>".format(
            project_name=project_name, spider_name=spider_name, spider_num=number)
        tds.append(aps)
    table = "".join(tds) if len(tds) else ""
    return table

```

它会根据传入的值进行计算，循环获取其中的爬虫项目名、爬虫名与爬虫数量，用 format 的方式将 HTML 与数值结合生成`<tr><td></td></tr>`结构的数据。然后在 Home 类的 render 中获取结果列表：

```
 def render_GET(self, request):
        pending, running, finished, average, shortest, \
        longest, projects, spiders, ranks, dps, \
        lpn, lsn, invoked_spider, un_invoked_spider, most_record = features(self)
        table = make_table(dps)

```

## 界面重构-首页页面拼接之 URL 跳转

在 Scrapyd 首页中，URL 跳转使用的是`<a href="/jobs">Jobs</a>`，考虑到权限验证的请求是需要携带正确的用户名、密码，所以需要根据访问者请求参数生成对应的 URL 跳转地址，既要考虑到用户未携带参数又要考虑用户携带参数，代码逻辑如下：

*   从 request 中获取用户名和密码
*   通过循环为每一个需要用到的 URL 地址加上用户名和密码，生成新的 URL 地址列表。
*   这样设置的好处是访问者只有在携带正确用户名和密码后才能一路畅通的跳转各个 URL 地址
*   如果使用了错误的用户名和密码或不填写用户名和密码，那么后续的 URL 跳转也将不会有用户名和密码信息，这样既保证了访问者使用时的流畅性又能够保证权限验证不会被绕过。

在`webtools.py`中新建函数`make_urls`:

```
def make_urls(hosts):
    urls = []
    for x in ["/", "/jobs", "/feature", "/documents"]:
        uri = "?un=%s&pwd=%s" % (hosts.get("un"), hosts.get("pwd"))
        # "http://" + hosts.get("host") + ":" + hosts.get("port") + x + uri 获取host中的ip会造成云服务器跳转失败
        url = x + uri
        urls.append(url)
    return urls

```

跟以往一样，在 Home 类中可以对`make_urls`的返回结果进行拆包使用了么？还不可以，`make_urls(host)`接收参数 host，从 host 信息中取出对应参数的用户名和密码。这里在`webtools.py`中再新建名为`host_information`的函数：

```
def host_information(request):
    host, port, un, pwd = request.host.host, str(request.host.port), "un", "pwd"
    if request.args.get(bytes(un, encoding="utf-8")) and request.args.get(bytes(pwd, encoding="utf-8")):
        username = str(request.args.get(bytes(un, encoding="utf-8"))[0], encoding="utf-8")
        password = str(request.args.get(bytes(pwd, encoding="utf-8"))[0], encoding="utf-8")
        return {"host": host, "port": port, "un": username, "pwd": password}
    return {"host": host, "port": port, "username": "", "pwd": ""}

```

其中的代码逻辑是从 request 请求信息中获取关键字`un`、`pwd`对应的值，并且从 request.host 可以直接获取 host 和 port 信息，最后以 dict 的形式将结果返回。同时也在代码中对请求参数进行了简化，因为配置文件中的参数设定是`auth_username`和`auth_password`,这里则限定为`un`和`pwd`。

通过上面的两个方法获得了动态的 URL， 在 website.py 中引入这两个方法并在 Home 类的 render 中使用：

```
from .webtools import  microsec_trunc, host_information, make_urls

 def render_GET(self, request):
        pending, running, finished, average, shortest, \
        longest, projects, spiders, ranks, dps, \
        lpn, lsn, invoked_spider, un_invoked_spider, most_record = features(self)
        table = make_table(dps)
        hosts = host_information(request)  # host以及auth信息
        home_uri, jobs_uri, feature_uri, documents_uri = make_urls(hosts)

```

URL 跳转地址需要在 HTML 的导航部分使用，由于导航的选中状态是用 class 属性来改变，所以 HTML 中导航部分的代码需要单独提取出来。然后将 template 目录中的`header.html`、`index.html`、`footer.html`进行拼接，在拼接时将所需要渲染的数据通过 format 的方式填充：

```
nav = """
    <ul>
    <li>
    <a href={home_uri} class="selected" id="home">
    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <polygon fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" points="12,2 3,10 3,23 9,23 9,16 15,16 15,23 21,23 21,10 " stroke-linejoin="miter"></polygon> </g></svg>
    Home</a>
    </li>
    <li>
    <a href={jobs_uri} class="item" id="jobs">
    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <polyline data-color="color-2" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" points=" 16,7 16,2 8,2 8,7 " stroke-linejoin="miter"></polyline> <rect x="1" y="7" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="22" height="15" stroke-linejoin="miter"></rect> <line fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" x1="5" y1="7" x2="5" y2="22" stroke-linejoin="miter"></line> <line fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" x1="19" y1="7" x2="19" y2="22" stroke-linejoin="miter"></line> </g></svg>
    Jobs</a>
    </li>
    <li>
    <a href={feature_uri} class="item" id="feature">
    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <rect x="1" y="1" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="22" height="22" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="5" y="5" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="14" y="5" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="5" y="14" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="14" y="14" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> </g></svg>
    ArtDoc</a>
    </li>
    </ul>
        """.format(home_uri=home_uri, jobs_uri=jobs_uri, feature_uri=feature_uri,
                   documents_uri=documents_uri)
header = HEADER_HTML.format(style_css=STYLE_CSS, reset_css=RESET_CSS, nav=nav)

index = INDEX_HTML.format(projects=dps, nop=pending, nor=running, nof=finished, average=average,
                      shortest=shortest, longest=longest, lpn=lpn, lsn=lsn, most_spider=most_record[0],
                      most_num=most_record[1], invoked_spider=",".join(invoked_spider),
                      un_invoked_spider=",".join(un_invoked_spider), table=table, ranks=ranks,
                      feature_uri=feature_uri)
footers = FOOTERS_HTML

```

而在 HTML 中则需要使用 format 占位符的写法将所需渲染的数据位置进行标记，例如填充爬虫调用情况数据：

```
<div class="show-data">
	<div class="show-data-middle">
	<div class="sdm-top">
		<p><span>被调用过的爬虫 : </span><span>{invoked_spider}</span></p>
		<p><span>被调用次数最多 : </span><span>{most_spider},{most_num}次</span></p>
		<p><span>未被调用的爬虫 : </span><span>{un_invoked_spider}</span></p>
	</div>
</div>

```

`index.html`中**其他需要数据填充的地方**按照上方 HTML 代码的 format 形式进行**填充**即可，否则后续数据渲染会报出异常。

最后，根据 CustomResource 的设定，将所需渲染的内容以 bytes 类型返回，所以整个 Home 类的完整代码如下：

```
class Home(CustomResource):

    def __init__(self, root, local_items):
        resource.Resource.__init__(self)
        self.root = root
        self.local_items = local_items

    @decorator_auth
    def render_GET(self, request):
        pending, running, finished, average, shortest, \
        longest, projects, spiders, ranks, dps, \
        lpn, lsn, invoked_spider, un_invoked_spider, most_record = features(self)

        table = make_table(dps)
        hosts = host_information(request)  # host以及auth信息
        home_uri, jobs_uri, feature_uri, documents_uri = make_urls(hosts)

        nav = """
                <ul>
                    <li>
                    <a href={home_uri} class="selected" id="home">
                    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <polygon fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" points="12,2 3,10 3,23 9,23 9,16 15,16 15,23 21,23 21,10 " stroke-linejoin="miter"></polygon> </g></svg>
                    Home</a>
                    </li>
                    <li>
                    <a href={jobs_uri} class="item" id="jobs">
                    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <polyline data-color="color-2" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" points=" 16,7 16,2 8,2 8,7 " stroke-linejoin="miter"></polyline> <rect x="1" y="7" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="22" height="15" stroke-linejoin="miter"></rect> <line fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" x1="5" y1="7" x2="5" y2="22" stroke-linejoin="miter"></line> <line fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" x1="19" y1="7" x2="19" y2="22" stroke-linejoin="miter"></line> </g></svg>
                    Jobs</a>
                    </li>
                    <li>
                    <a href={feature_uri} class="item" id="feature">
                    <svg class="nc-icon outline" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="24px" height="24px" viewBox="0 0 24 24"><g transform="translate(0, 0)"> <rect x="1" y="1" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="22" height="22" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="5" y="5" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="14" y="5" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="5" y="14" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> <rect data-color="color-2" x="14" y="14" fill="none" stroke="#4a5261" stroke-width="2" stroke-linecap="square" stroke-miterlimit="10" width="5" height="5" stroke-linejoin="miter"></rect> </g></svg>
                    ArtDoc</a>
                    </li>
                </ul>
                    """.format(home_uri=home_uri, jobs_uri=jobs_uri, feature_uri=feature_uri,
                               documents_uri=documents_uri)
        header = HEADER_HTML.format(style_css=STYLE_CSS, reset_css=RESET_CSS, nav=nav)

        index = INDEX_HTML.format(projects=dps, nop=pending, nor=running, nof=finished, average=average,
                                  shortest=shortest, longest=longest, lpn=lpn, lsn=lsn, most_spider=most_record[0],
                                  most_num=most_record[1], invoked_spider=",".join(invoked_spider),
                                  un_invoked_spider=",".join(un_invoked_spider), table=table, ranks=ranks,
                                  feature_uri=feature_uri)
        footers = FOOTERS_HTML
        self.content = str_to_bytes(header + index + footers)  # return value need bytes
        return self.content

```

## 小结

需要注意的是，由于权限装饰器的存在，所以 URL 跳转需要额外处理。

前面几个小节完成了 ScrapydArt 的首页功能编写与 HTML 页面拼接，下面将对 Jobs 页面进行重构。