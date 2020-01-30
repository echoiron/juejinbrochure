# 编写兼容 HTML 与 JSON 的视图类

## 改造原理

### 为什么要改写

后面的章节中，我们将会为 Scrapyd 加上**访问权限访问控制**的功能。而权限既要能够在请求 HTML 资源的时候发挥作用，也要在请求 JSON 资源的时候发挥作用，当你提交正确的验证信息，它会让你通过验证并返回正确的结果（这个结果有可能是 HTML 也有可能是 JSON ），所以这里就需要编写一个兼容 HTML 与 JSON 的视图类，并且对现有的 Home、Jobs 和其他 API 进行些许调整，以达到目的。

### 原理

主要是利用 Python 类的**继承和多态**这两个特性

> 继承可以把父类的所有功能都直接拿过来，这样就不必从 0 做起，子类只需要新增自己特有的方法，也可以把父类不适合的方法覆盖重写；

> 有了继承，才能有多态。在调用类实例方法的时候，尽量把变量视作父类。这样，所有子类类型都可以正常被接收；

在这里，笔者需要自定义一个类，它继承`WsResource`和`resource.Resource`这两个类，这样即可拥有他们的特性，然后笔者通过重写 render 就可以达到目的。

## 改造代码和步骤

### 自定义类与继承

在`webservice.py`中，笔者创建了一个名为`CustomResource`的类，它继承自 JsonResource 和 resource.Resource。由于 Python 中类继承的`__mro__`顺序存在，所以继承顺序不能随意，必须是`JsonResource` 在前面，代码如下：

```
from twisted.web import resource

class CustomResource(JsonResource, resource.Resource):

```

上面也提到过，通过继承的方式，我们可以让新的子类具备父类的方法及特性，还可以通过重写或者新增方法来增加新特性。

### 重写 render 与 init

因为继承了两个类，根据 Python 的`__mro__`顺序，如果不重写的话是有可能出现问题的，所以笔者需要先重写`__init__`方法

```
def __init__(self, root):
    JsonResource.__init__(self)
    self.root = root

```

其实就是直接使用了 JsonResource 的`__init__`方法。

接着重写 render，以实现两种视图数据的兼容

```
def render(self, txrequest):
    try:
        return JsonResource.render(self, txrequest).encode('utf-8')
    except Exception:
        return self.content

```

GIF 动态图中完整的演示了 `CustomResource` 类的编码过程：

![](https://user-gold-cdn.xitu.io/2018/10/11/166627302981415b?w=1365&h=862&f=gif&s=2830332)

通过`try except`的方式来实现，优先返回 json 格式数据，如果出现错误则返回`self.content`（原 JsonResource 在 except 中返回错误信息）。

### 新视图类的使用

编写完自定义的视图类后，笔者将在`website.py`中使用，首先将它引入

```
from .webservice import CustomResource

```

接着将 Home 方法的父类由 resource.Resource 改为`CustomResource`

```
class Home(CustomResource)

```

再在 render\_GET 中将想要返回的页面赋值给变量`self.content`，最后将`self.content`返回

```
def render_GET(self, request):
    ……
    self.content = s.encode('utf-8')
    return self.content

```

![](https://user-gold-cdn.xitu.io/2018/10/11/166627fda1053072?w=1170&h=781&f=gif&s=3507135)

> 小提示：如果是 JSON 视图，直接继承 CustomResource 即可，Return 处无需改动。

这样就完成了，当我们访问 Home 类的时候，它会根据父类 render 的设定优先返回 JSON 数据，现在是非 JSON 数据，所以会报错。进入 except 流程并返回 self.content，这样就完成了兼容 HTML 视图与 JSON 视图的自定义类`CustomResource`的编写。

在`Jobs`和其他 API 上的使用方法与在`Home`上的的使用方法是一样的。

## 小结

本节先理解改写原因和原理将逻辑整理清晰，然后通过实际的编码，实现了兼容 HTML 与 JSON 的视图类，为后面权限访问控制以及 Scrapyd 界面改造打下了基础。