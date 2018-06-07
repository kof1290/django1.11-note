# Django基础知识点整理

`django1.11`

> ## 目录结构如下

- [Url](#url)
    - project url
    - app url
        - RegEx
        - args
        - kwargs

- [View](#view)
    - def view
        - HttpResponse
        - render
        - HttpResponseRedirect
        - redirect
    - class based view
        - Base
            - View
            - TemplateView
            - RedirectView
        - Display
            - ListView
            - DetailView
        - Edit
            - FormView
            - CreateView
            - UpdateView
            - DeleteView
- [Template](#template)
    - Variables
    - Template inheritance
    - Custom tag and filter libraries
        - tags
        - apptags
        - filters
    - context processors
    - static
- [Model](#model)
    - define and use
    - Fields
        - relationship `没有多对一关系`
            - OneToOne
            - OneToMany
            - ManyToMany
    - Meta options
    - Model methods
    - Model inheritance
    - Making queries


---

#### Url
##### project url

在project的`urls.py`文件中使用`url(r'^myapp/',include('myapp.urls')),`来包含你的`app url`

##### app url
> RegEx
>> url通过正则表达式匹配 ^ 作为一个url开始的标识符，$作为url的结束标识符

> args
>> url中的位置参数 `url('^articles/([0-9]{4})/([0-9]{2})$', ArticleListView.as_view()),`，url如`articles/2018/01/` 相当于调用cbv中的方法`get(request, '2018', '01')`

> kwargs
>> url中的关键字参数 使用最多的方式 `url(r'^line/(?P<pk>[0-9]+)$', views.EquView.as_view(), name='equ'),`,url如`line/?pk=1/`调用的就是EquView中的`get(request,pk=1)`，这里的关键词pk是singelobjectmixin中的默认值，需要更改可以用`pk_url_kwarg='pk'`属性改掉

---

#### View
> Django使用`request`和`response`对象在系统中传递信息，当发生一个请求时，Django会创建一个`HttpRequest`对象也就是`request`，然后会去找对应的视图把`HttpRequest`对象作为第一个参数传递给视图函数。每个视图都会负责返回一个`HTTPResponse`对象。

##### def view


> HttpResponse
>> from django.http import HttpResponse

    def index(request):
        return HttpResponse('hello world!')

这是个最简单的视图函数，所有的教程也都是从这里开始的，但是我们发现了，返回字符串多费劲啊，没事我们有`render`函数

> render
>> from django.shortcuts import render

    # views.py
    def index(request):
        text = 'hello world!'
        return render(request, 'index.html',{'text':text})
        # local()可以把所有变量一同传递过去，而不需要一个个写，模板中的变量名同定义的名字
        # return render(request, 'index.html', local())
    # index.html
    {{ text }}

> HttpResponseRedirect
>> from django.http import HttpResponseRedirect

django的重定向使用`HttpResponseRedirect`

    from django.http import HttpResponseRedirect

    def my_redirect(request):
        ...
        return HttpResponseRedirect("/index/")

也可以通过url的name解析

    # urls.py
    from django.conf.urls import url
    from myapp import views

    urlpatterns = [
        url(r'^index/$', views.myview, name='index'),
    ]

    # views.py
    from django.http import HttpResponseRedirect
    from django.core.urlresolvers import reverse

    def myview(request):
        ...
        return HttpResponseRedirect(reverse('index'))
        # 带传值的方式
        # 一个值
        # value = 你需要带的值
        # return HttpResponseRedirect(reverse('index', args=(value))
        # 多个值
        # return HttpResponseRedirect(reverse('index', kwargs={'page':page,'id':id}))

> redirect
>> from django.shortcuts import redirect

redirect 是HttpResponseRedirect的高级版本，用法基本一致
`redirect(to, permanent=False, *args, **kwargs)`他集成了一个永久重定向，可以这么理解，永久重定向返回`301`，一般重定向返回`302`

[request-response](https://docs.djangoproject.com/en/1.11/ref/request-response/)里面都是一些基础的函数，许多高级函数都是基于此包装的

[shortcut](https://docs.djangoproject.com/en/1.11/topics/http/shortcuts/)里面包含了render和redirect以及很多其他的函数，需要可以去看看


#### class based view
##### base
> View
>> View是所有类视图的父类，其中定义了允许的http_method，以及两个主要方法，`as_view()`和`dispatch()`

>> `as_view()`是View的主要入口，用于处理`request`和`response`

>> `dispatch()`是用来给那些http请求分派任务的，可以在一个请求生效之前进行一些操作，栗如可以在一个get请求生效之前判断此用户是否具有权限，然后返回`403 Forbidden`


    class View(object):
        """
        Intentionally simple parent class for all views. Only implements
        dispatch-by-method and simple sanity checking.
        """

        http_method_names = ['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace']

        def __init__(self, **kwargs):
            """
            Constructor. Called in the URLconf; can contain helpful extra
            keyword arguments, and other things.
            """
            # Go through keyword arguments, and either save their values to our
            # instance, or raise an error.
            for key, value in six.iteritems(kwargs):
                setattr(self, key, value)

        @classonlymethod
        def as_view(cls, **initkwargs):
            """
            Main entry point for a request-response process.
            """
            for key in initkwargs:
                if key in cls.http_method_names:
                    raise TypeError("You tried to pass in the %s method name as a "
                                    "keyword argument to %s(). Don't do that."
                                    % (key, cls.__name__))
                if not hasattr(cls, key):
                    raise TypeError("%s() received an invalid keyword %r. as_view "
                                    "only accepts arguments that are already "
                                    "attributes of the class." % (cls.__name__, key))

            def view(request, *args, **kwargs):
                self = cls(**initkwargs)
                if hasattr(self, 'get') and not hasattr(self, 'head'):
                    self.head = self.get
                self.request = request
                self.args = args
                self.kwargs = kwargs
                return self.dispatch(request, *args, **kwargs)
            view.view_class = cls
            view.view_initkwargs = initkwargs

            # take name and docstring from class
            update_wrapper(view, cls, updated=())

            # and possible attributes set by decorators
            # like csrf_exempt from dispatch
            update_wrapper(view, cls.dispatch, assigned=())
            return view

        def dispatch(self, request, *args, **kwargs):
            # Try to dispatch to the right method; if a method doesn't exist,
            # defer to the error handler. Also defer to the error handler if the
            # request method isn't on the approved list.
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
            return handler(request, *args, **kwargs)

        def http_method_not_allowed(self, request, *args, **kwargs):
            logger.warning(
                'Method Not Allowed (%s): %s', request.method, request.path,
                extra={'status_code': 405, 'request': request}
            )
            return http.HttpResponseNotAllowed(self._allowed_methods())

        def options(self, request, *args, **kwargs):
            """
            Handles responding to requests for the OPTIONS HTTP verb.
            """
            response = http.HttpResponse()
            response['Allow'] = ', '.join(self._allowed_methods())
            response['Content-Length'] = '0'
            return response

        def _allowed_methods(self):
            return [m.upper() for m in self.http_method_names if hasattr(self, m)]

> TemplateView
>> 他继承了`TemplateResponseMixin`、`ContextMixin`、`View`

>> `TemplateResponseMixin`的作用就是`render`一个页面

    class TemplateResponseMixin(object):
        """
        A mixin that can be used to render a template.
        """
        template_name = None
        template_engine = None
        response_class = TemplateResponse
        content_type = None

        def render_to_response(self, context, **response_kwargs):
            """
            Returns a response, using the `response_class` for this
            view, with a template rendered with the given context.

            If any keyword arguments are provided, they will be
            passed to the constructor of the response class.
            """
            response_kwargs.setdefault('content_type', self.content_type)
            return self.response_class(
                request=self.request,
                template=self.get_template_names(),
                context=context,
                using=self.template_engine,
                **response_kwargs
            )

        def get_template_names(self):
            """
            Returns a list of template names to be used for the request. Must return
            a list. May not be called if render_to_response is overridden.
            """
            if self.template_name is None:
                raise ImproperlyConfigured(
                    "TemplateResponseMixin requires either a definition of "
                    "'template_name' or an implementation of 'get_template_names()'")
            else:
                return [self.template_name]

>> `ContextMixin`是传递一些额外的变量

    class ContextMixin(object):
        """
        A default context mixin that passes the keyword arguments received by
        get_context_data as the template context.
        """

        def get_context_data(self, **kwargs):
            if 'view' not in kwargs:
                kwargs['view'] = self
            return kwargs

>> 由此可知TemplateView的作用是渲染一个模板，并且可以传递变量


> RedirectView
>> 类视图中的重定向，没有什么特别的地方

##### display
> ListView

> 代码结构

    # list.py
    class MultipleObjectMixin(ContextMixin):
    ...
    class BaseListView(MultipleObjectMixin, View):
    """
    A base view for displaying a list of objects.
    """
    ...
    class MultipleObjectTemplateResponseMixin(TemplateResponseMixin):
    ...
    class ListView(MultipleObjectTemplateResponseMixin, BaseListView):
    ...


栗子

    # views.py

    from django.views.generic.list import ListView
    from django.utils import timezone

    from articles.models import Article

    class ArticleListView(ListView):

        model = Article

        def get_context_data(self, **kwargs):
            context = super(ArticleListView, self).get_context_data(**kwargs)
            context['now'] = timezone.now()
            return context

    # urls.py

    from django.conf.urls import url

    from article.views import ArticleListView

    urlpatterns = [
        url(r'^$', ArticleListView.as_view(), name='article-list'),
    ]

    # article_list.html
    <h1>Articles</h1>
    <ul>
    {% for article in object_list %}
        <li>{{ article.pub_date|date }} - {{ article.headline }}</li>
    {% empty %}
        <li>No articles yet.</li>
    {% endfor %}
    </ul>




> DetailView

> 代码结构

    # detail.py
    class SingleObjectMixin(ContextMixin):
    ...
    class BaseDetailView(SingleObjectMixin, View):
    """
    A base view for displaying a single object
    """
    ...
    class SingleObjectTemplateResponseMixin(TemplateResponseMixin):
    ...
    class DetailView(SingleObjectTemplateResponseMixin, BaseDetailView):
    ...

栗子

    # views.py

    from django.views.generic.detail import DetailView
    from django.utils import timezone

    from articles.models import Article

    class ArticleDetailView(DetailView):

        model = Article

        def get_context_data(self, **kwargs):
            context = super(ArticleDetailView, self).get_context_data(**kwargs)
            context['now'] = timezone.now()
            return context

    # urls.py

    from django.conf.urls import url

    from article.views import ArticleDetailView

    urlpatterns = [
        url(r'^(?P<slug>[-\w]+)/$', ArticleDetailView.as_view(), name='article-detail'),
    ]

    # article_detail.html

    <h1>{{ object.headline }}</h1>
    <p>{{ object.content }}</p>
    <p>Reporter: {{ object.reporter }}</p>
    <p>Published: {{ object.pub_date|date }}</p>
    <p>Date: {{ now|date }}</p>

> 栗子中的模板名字，都是默认值，`ListView`的模板为`model_list.html`,同理`DetailView`的模板为`model_detail.html`，不过一般情况我们都会使用`template_name`来指明使用的模板

> 由于`ListView`和`DetailView`都是基于`Base`里面的`ContextMixin`,`View`,`TemplateResponseMixin`这三个类，那么理解他们的作用就很简单了，源代码中也解释了作用

>> `ListView`用于`displaying a list of objects`

>> `DetailView`用于`displaying a single object`


##### edit
> FormView

> 代码结构

    # edit.py

    class FormMixin(ContextMixin):
    """
    A mixin that provides a way to show and handle a form in a request.
    """

    class ProcessFormView(View):
    """
    A mixin that renders a form on GET and processes it on POST.
    """

    class BaseFormView(FormMixin, ProcessFormView):
    """
    A base view for displaying a form.
    """

    class FormView(TemplateResponseMixin, BaseFormView):
    """
    A view for displaying a form, and rendering a template response.
    """

栗子

    # forms.py

    from django import forms

    class ContactForm(forms.Form):
        name = forms.CharField()
        message = forms.CharField(widget=forms.Textarea)

        def send_email(self):
            # send email using the self.cleaned_data dictionary
            pass

    # views.py
    from myapp.forms import ContactForm
    from django.views.generic.edit import FormView

    class ContactView(FormView):
        template_name = 'contact.html'
        form_class = ContactForm
        success_url = '/thanks/'

        def form_valid(self, form):
            # This method is called when valid form data has been POSTed.
            # It should return an HttpResponse.
            form.send_email()
            return super(ContactView, self).form_valid(form)

    # contact.html
    <form action="" method="post">{% csrf_token %}
        {{ form.as_p }}
        <input type="submit" value="Send message" />
    </form>

> CreateView

> 代码结构

    # edit.py

    class FormMixin(ContextMixin):
    """
    A mixin that provides a way to show and handle a form in a request.
    """

    class SingleObjectMixin(ContextMixin):
    """
    Provides the ability to retrieve a single object for further manipulation.
    """

    class ModelFormMixin(FormMixin, SingleObjectMixin):
    """
    A mixin that provides a way to show and handle a modelform in a request.
    """

    class ProcessFormView(View):
    """
    A mixin that renders a form on GET and processes it on POST.
    """

    class BaseCreateView(ModelFormMixin, ProcessFormView):
    """
    Base view for creating an new object instance.
    Using this base class requires subclassing to provide a response mixin.
    """

    class SingleObjectTemplateResponseMixin(TemplateResponseMixin):

    class CreateView(SingleObjectTemplateResponseMixin, BaseCreateView):
    """
    View for creating a new object instance,
    with a response rendered by template.
    """

栗子

    # views.py

    from django.views.generic.edit import CreateView
    from myapp.models import Author

    class AuthorCreate(CreateView):
        model = Author
        fields = ['name']
        # 当然你可以用template_name来指明模板名字

    # author_form.html
    <form action="" method="post">{% csrf_token %}
        {{ form.as_p }}
        <input type="submit" value="Save" />
    </form>

> UpdateView

> 代码结构

    # edit.py

    class FormMixin(ContextMixin):
    """
    A mixin that provides a way to show and handle a form in a request.
    """

    class SingleObjectMixin(ContextMixin):
    """
    Provides the ability to retrieve a single object for further manipulation.
    """

    class ModelFormMixin(FormMixin, SingleObjectMixin):
    """
    A mixin that provides a way to show and handle a modelform in a request.
    """

    class ProcessFormView(View):
    """
    A mixin that renders a form on GET and processes it on POST.
    """

    class BaseUpdateView(ModelFormMixin, ProcessFormView):
    """
    Base view for updating an existing object.
    Using this base class requires subclassing to provide a response mixin.
    """

    class SingleObjectTemplateResponseMixin(TemplateResponseMixin):

    class UpdateView(SingleObjectTemplateResponseMixin, BaseUpdateView):
    """
    View for updating an object,
    with a response rendered by template.
    """

栗子

    # views.py
    from django.views.generic.edit import UpdateView
    from myapp.models import Author

    class AuthorUpdate(UpdateView):
        model = Author
        fields = ['name']
        template_name_suffix = '_update_form' # 自己定义模板后缀

    # author_update_form.html
    <form action="" method="post">{% csrf_token %}
        {{ form.as_p }}
        <input type="submit" value="Update" />
    </form>

> DeleteView

> 代码结构

    # edit.py

    class DeletionMixin(object):
    """
    A mixin providing the ability to delete objects
    """

    class BaseDetailView(SingleObjectMixin, View):
    """
    A base view for displaying a single object
    """

    class SingleObjectMixin(ContextMixin):
    """
    Provides the ability to retrieve a single object for further manipulation.
    """

    class BaseDeleteView(DeletionMixin, BaseDetailView):
    """
    Base view for deleting an object.
    Using this base class requires subclassing to provide a response mixin.
    """

    class SingleObjectTemplateResponseMixin(TemplateResponseMixin):

    class DeleteView(SingleObjectTemplateResponseMixin, BaseDeleteView):
    """
    View for deleting an object retrieved with `self.get_object()`,
    with a response rendered by template.
    """

栗子

    # views.py
    from django.views.generic.edit import DeleteView
    from django.urls import reverse_lazy
    from myapp.models import Author

    class AuthorDelete(DeleteView):
        model = Author
        success_url = reverse_lazy('author-list')

    # author_confirm_delete.html
    <form action="" method="post">{% csrf_token %}
        <p>Are you sure you want to delete "{{ object }}"?</p>
        <input type="submit" value="Confirm" />
    </form>


> `edit`中对应了我们常用的增改删操作，均继承了`SingleObjectMixin`，对单一`object`执行创建、更新、删除的操作
>> 上面的栗子只是最简单的实现，真正应用需要额外的学习 [请点击我查看详细](https://docs.djangoproject.com/en/1.11/ref/class-based-views/generic-editing/)

---
#### Template
##### Virables
> 简单来说就像`{{ variable }}`这样子，使用`.`符号访问变量的属性

栗子

    {% for k, v in defaultdict.iteritems %}
        Do something with k and v here...
    {% endfor %}

变量配合一些自带的`tag`和`filter`在模板里写上你的逻辑很好用，后面会做介绍，详细可以[点击这里](https://docs.djangoproject.com/en/1.11/ref/templates/builtins/)

##### Template inheritance
> 模板继承就是把页面理解为拼图，不同的拼图拼出来不同的页面
>> 我们需要一个`base`页面，然后使用`{% block name %} {% endblock %}`这样的标识来定义每一块拼图，拼图越多越好，有些拼图，类似于`footer`你可以使用`{% include 'footer.html' %}`来把他引入到你的`base`页面，在拼图页面需要使用`{% extend 'base.html' %}`放在页面最顶端表明继承的这个页面

栗子

    <!--base.html-->

    <!DOCTYPE html>
    <html lang="en">
    <head>
        <link rel="stylesheet" href="style.css" />
        <title>{% block title %}My amazing site{% endblock %}</title>
    </head>

    <body>
        <div id="sidebar">
            <!--子页面没覆盖sidebar就会展示这个-->
            {% block sidebar %}
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/blog/">Blog</a></li>
            </ul>
            {% endblock %}
        </div>

        <div id="content">
            {% block content %}{% endblock %}
        </div>
        {% include 'footer.html' %}
    </body>
    </html>

    <!--child.html-->
    {% extends "base.html" %}

    {% block title %}My amazing blog{% endblock %}

    {% block content %}
    {% for entry in blog_entries %}
        <h2>{{ entry.title }}</h2>
        <p>{{ entry.body }}</p>
    {% endfor %}
    {% endblock %}

> 模板继承是非常简单灵活的，可以使用多重继承，官方推荐的是三层结构
>> `base.html`-->`base_news.html`/`base_sports.html`-->`child.html`，当然这是因为`Django`出自新闻网站咯，不过适合的才是最好的

##### Custom tag and filter libraries
> 这里是`Template`的重头戏，逻辑判断、遍历和一些实用的函数都在这里面，全部罗列太多了，捡一些常用的吧
>> Tags
>>> `csrf_token`用来保护`CSRF`（跨站请求伪造），适用于表单提交，[详细了解一下](https://docs.djangoproject.com/en/1.11/ref/csrf/)

    <form action="" method="post">{% csrf_token %}

>>> `for`,`for...empty` 遍历

    {# for #}
    <ul>
    {% for athlete in athlete_list %}
        <li>{{ athlete.name }}</li>
    {% endfor %}
    </ul>

    {% for x, y in points %}
        There is a point at {{ x }},{{ y }}
    {% endfor %}

    {% for key, value in data.items %}
        {{ key }}: {{ value }}
    {% endfor %}

    {# for...empty #}
    <ul>
    {% for athlete in athlete_list %}
        <li>{{ athlete.name }}</li>
    {% empty %}
        <li>Sorry, no athletes in this list.</li>
    {% endfor %}
    </ul>


>>> `if` 判断

    {% if athlete_list %}
        Number of athletes: {{ athlete_list|length }}
    {% elif athlete_in_locker_room_list %}
        Athletes should be out of the locker room soon!
    {% else %}
        No athletes.
    {% endif %}

    {# use Boolean operators #}
    {% if athlete_list and coach_list %}
        Both athletes and coaches are available.
    {% endif %}

    {% if not athlete_list %}
        There are no athletes.
    {% endif %}

    {% if athlete_list or coach_list %}
        There are some athletes or some coaches.
    {% endif %}

    {% if not athlete_list or coach_list %}
        There are no athletes or there are some coaches.
    {% endif %}

    {% if athlete_list and not coach_list %}
        There are some athletes and absolutely no coaches.
    {% endif %}

    {% if athlete_list and coach_list or cheerleader_list %}

>>>> 还有这些操作符 `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`, `is`, and `is not`

>>> `include` 加载一个`template`并使用当前的`context`渲染他，每一个`include`的页面都是`独立`的

    {% include "name_snippet.html" %}

    {# name_snippet.html #}
    {{ greeting }}, {{ person|default:"friend" }}!

    {# 可以给模板传值 #}
    {% include "name_snippet.html" with person="Jane" greeting="Hello" %}

    {# 如果不想使用全文的context，可以使用only来指明 #}
    {% include "name_snippet.html" with greeting="Hi" only %}


>>> `load` 加载`tags`和`filters`

    {# 加载全部 #}
    {% load somelibrary package.otherlibrary %}

    {# 加载某几个 #}
    {% load foo bar from somelibrary %}



>>> `url` 路径和最开始那部分url对应

    {# 位置参数 #}
    {% url 'some-url-name' v1 v2 %}

    {# 关键字参数 #}
    {% url 'some-url-name' arg1=v1 arg2=v2 %}

一个完整的栗子

    # app/urls.py
    ('^client/([0-9]+)/$', app_views.client, name='app-views-client')

    # project/urls.py
    ('^clients/', include('project_name.app_name.urls'))

    {#  一个url #}
    {% url 'app-views-client' client.id %}


最终会渲染成这个样子`/clients/client/123/`


>>> `with` 把变量换个名字，作用域仅限于这个block

    {% with total=business.employees.count %}
        {{ total }} employee{{ total|pluralize }}
    {% endwith %}

    {% with alpha=1 beta=2 %}
        ...
    {% endwith %}

    {% with business.employees.count as total %}

    {% endwith %}


>> apptags
>>> 自定义tag的目的就是可以在模板中载入一些`通用数据`，例如`header`、`sidebar`、`tags`中的数据。

目录结构推荐这样

    polls/
        __init__.py
        models.py
        templatetags/
            __init__.py
            poll_extras.py
        views.py

这样使用

    {# example.html #}
    {% load poll_extras %}
    {% sidebar as sdata %}
    ...

    # poll_extras.py
    from django import template
    from ..models import ProdLine

    register = template.Library()
    @register.simple_tag
    def sidebar():
        return ProdLine.objects.all()

>> Filters

>>>`add` “加法运算”

    {# If value is 4, then the output will be 6. #}
    {{ value|add:"2" }}


    {# first is [1, 2, 3] and second is [4, 5, 6], then the output will be [1, 2, 3, 4, 5, 6]. #}
    {{ first|add:second }}

>>>`cut` “减法运算”

    {# 去掉空格 If value is "String with spaces", the output will be "Stringwithspaces". #}
    {{ value|cut:" " }}


>>>`date` 格式化日期 [查看详细](https://docs.djangoproject.com/en/1.11/ref/templates/builtins/#date)

    {# If value is a datetime object (e.g., the result of datetime.datetime.now()), the output will be the string 'Wed 09 Jan 2008'. #}
    {{ value|date:"D d M Y" }}



>>>`default` 设定默认值

    {{ value|default:"nothing" }}

>>>`dictsort` 排序

    {# 按name排序 #}
    {{ value|dictsort:"name" }}

value为

    [
        {'name': 'zed', 'age': 19},
        {'name': 'amy', 'age': 22},
        {'name': 'joe', 'age': 31},
    ]

输出为

    [
        {'name': 'amy', 'age': 22},
        {'name': 'joe', 'age': 31},
        {'name': 'zed', 'age': 19},
    ]


>>>`join` 类似于 str.join(list)

    {# If value is the list ['a', 'b', 'c'], the output will be the string "a // b // c". #}
    {{ value|join:" // " }}

>>>`length` 输出`list`或者`str`的长度

    {# If value is [1,2,3,4] or "abcd", the output will be 4 #}
    {{ value|length }}

>>>`make_list` 把`string`或者`integer`转换成`list`

    {{ value|make_list }}

>>>`pluralize` 复数形式，很实用的一个过滤器，默认加`s`，当然有`es`的复数形式需要自己定义

    You have {{ num_messages }} message{{ num_messages|pluralize }}.

    You have {{ num_walruses }} walrus{{ num_walruses|pluralize:"es" }}.



>>>`random` list中的随机一项   ['a', 'b', 'c', 'd']--> 'b'

    {{ value|random }}

>>>`safe` 标记为安全的，这样就不会自动转义，在js中使用变量时会用到

    {{ var|safe }}

>>>`slice` 切片操作`slice(start, stop[, step])`，语法可以去这里[温习一下](http://www.diveintopython3.net/native-datatypes.html#slicinglists)

    {{ some_list|slice:":2" }}

还有很多过滤器，可以去[这里看](https://docs.djangoproject.com/en/1.11/ref/templates/builtins/#built-in-filter-reference)

更多关于自定义tag和filter的[看这里](https://docs.djangoproject.com/en/1.11/howto/custom-template-tags/)


##### context processors
> 这部分是关于模板上下文处理器，就是一些自带的变量，当然前提是你激活了特定的`processors`，就会有特定的变量添加到当前的`context`中

在`settings.py`文件中的`TEMPLATES`的`context_processors`

    TEMPLATES = [
        {
            'BACKEND': 'django.template.backends.django.DjangoTemplates',
            'DIRS': [os.path.join(BASE_DIR, 'templates')],
            'APP_DIRS': True,
            'OPTIONS': {

                'context_processors': [
                    'django.template.context_processors.debug',
                    'django.template.context_processors.request',
                    'django.contrib.auth.context_processors.auth',
                    'django.contrib.messages.context_processors.messages',
                ],
            },
        },
    ]

>> `django.template.context_processors.debug` 调试信息，一些报错的详细信息和sql语句
>>> 包含两个变量 `debug`返回`True`或者`False` 和`sql_queries` 当前的sql语句

>> `django.template.context_processors.request`
>>> 就一个变量 `request` 就是当前的[HttpRequest](https://docs.djangoproject.com/en/1.11/ref/request-response/#django.http.HttpRequest)，`request.path`就是当前的路径，不包括域名和端口号 栗如`/music/bands/the_beatles/`


>> `django.contrib.auth.context_processors.auth` 当前用户和用户的权限信息
>>> `user`和`perms`

    def auth(request):
        """
        Returns context variables required by apps that use Django's authentication
        system.

        If there is no 'user' attribute in the request, uses AnonymousUser (from
        django.contrib.auth).
        """
        if hasattr(request, 'user'):
            user = request.user
        else:
            from django.contrib.auth.models import AnonymousUser
            user = AnonymousUser()

        return {
            'user': user,
            'perms': PermWrapper(user),
        }

>> `django.contrib.messages.context_processors.messages`
>>> [消息机制](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/)包含`messages`和 `DEFAULT_MESSAGE_LEVELS`，在视图和模板中使用[了解一下](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/#using-messages-in-views-and-templates)

默认就是这四个处理器，更多相关的内容[点击我](https://docs.djangoproject.com/en/1.11/ref/templates/api/#built-in-template-context-processors)

##### static
网络上有很多博客有介绍如何配置静态资源`css`、`js`、`image`，但是有的会有很多坑导致看的头昏，这里[有个不错的](https://blog.csdn.net/sinat_21302587/article/details/74059078)如果你还不了解怎么配置就去看看吧


---

#### Model
Django ORM

##### define and use

> 最普通的一个模型可能是下面这样

    from django.db import models

    class Person(models.Model):
        # fields
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=30)

        # def
        def __str__(self):
            return self.first_name

        # meta
        class Meta:
            ordering = ['first_name']

> 当然定义完要在`settings.py`里告诉django我要使用这个`model`，像下面这样添加上`app_name`，最后要`manage.py makemigrations`再`manage.py migrate`

    INSTALLED_APPS = [
        #...
        'myapp',
        #...
    ]

##### Fields

> 字段是`model`中最重要并且是必须的部分

    from django.db import models

    class Musician(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)
        instrument = models.CharField(max_length=100)

    class Album(models.Model):
        artist = models.ForeignKey(Musician, on_delete=models.CASCADE)
        name = models.CharField(max_length=100)
        release_date = models.DateField()
        num_stars = models.IntegerField()

>> Field types
>>> 每个字段都对应数据库中的一列，`column type`每一列的类型我们需要指明，例如一些经常用到的`CharField` `IntegerField` `TextField` `URLField`等等，更多`types`[见这里](https://docs.djangoproject.com/en/1.11/ref/models/fields/#model-field-types)

>> Field options
>>> 指明字段类型之后还要对这个字段做一些限制，例如是否可以为空`null`（默认为`False`），注意区分`null`和`blank`的区别，后者一般应用在`form`表单声明此字段是否是必选项。`choice`给某个字段一些可选项，`primary_key`表明此字段为主键，如果没有声明主键，则会自动添加一个自增长主键`id`，`unique`声明字段是为唯一的默认值为`False`，`verbose_name`很有用，因为在展示的时候显示字段名很蠢。更多`options`[见这里](https://docs.djangoproject.com/en/1.11/ref/models/fields/#field-options)

>> Relationships
>>> 有三种关系`many-to-one` `many-to-many` `one-to-one`
>>>> `many-to-one`，定义一个多对一的关系需要用到`ForeignKey`

    from django.db import models

    class Manufacturer(models.Model):
        # ...
        pass

    # 多台汽车对应一个工厂
    class Car(models.Model):
        # 建议外键的字段名为模型的小写，当然自己定义也是完全可以的
        manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
        # ...

>>>> `many-to-many`，定义一个多对多关系需要用到`ManyToManyField`

    from django.db import models

    class Topping(models.Model):
        # ...
        pass

    # 一种披萨可以有多种料，一种料可以用在多个披萨上
    class Pizza(models.Model):
        # ...
        # 建议字段名使用关联模型的小写复数，当然只是建议而已
        # 多对多的关系放在哪个模型中需要根据实际情况，比如一种披萨可以加很多料
        toppings = models.ManyToManyField(Topping)

在一般的情况下上面这种结构就可以解决了，但是如果需要联系起来两个模型的数据，就像`人->乐团<-乐团成员`，就需要用到`through`，弄一个中间模型联系起来另外两个模型

    from django.db import models

    class Person(models.Model):
        name = models.CharField(max_length=128)

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    # 中间模型
    class Group(models.Model):
        name = models.CharField(max_length=128)
        members = models.ManyToManyField(Person, through='Membership')

        def __str__(self):              # __unicode__ on Python 2
            return self.name

    class Membership(models.Model):
        person = models.ForeignKey(Person, on_delete=models.CASCADE)
        group = models.ForeignKey(Group, on_delete=models.CASCADE)
        date_joined = models.DateField()
        invite_reason = models.CharField(max_length=64)

>>> `one-to-one` 一对一关系需要用到`OneToOneField`

    from django.db import models

    class Place(models.Model):
        name = models.CharField(max_length=50)
        address = models.CharField(max_length=80)

        def __str__(self):
            return "%s the place" % self.name

    class Restaurant(models.Model):
        place = models.OneToOneField(
            Place,
            on_delete=models.CASCADE,
            primary_key=True,
        )
        serves_hot_dogs = models.BooleanField(default=False)
        serves_pizza = models.BooleanField(default=False)

        def __str__(self):
            return "%s the restaurant" % self.place.name

    class Waiter(models.Model):
        restaurant = models.ForeignKey(Restaurant, on_delete=models.CASCADE)
        name = models.CharField(max_length=50)

        def __str__(self):
            return "%s the waiter at %s" % (self.name, self.restaurant)

三种关系的定义更多内容参考
[many-to-one](https://docs.djangoproject.com/en/1.11/topics/db/examples/many_to_one/) [many-to-many](https://docs.djangoproject.com/en/1.11/topics/db/examples/many_to_many/) [one-to-one](https://docs.djangoproject.com/en/1.11/topics/db/examples/one_to_one/)


##### Meta options
模型的一个内置类，主要作用是给模型一些额外的定义

> abstract 如果为`True`这个模型就会变成一个抽象模型，一般在模型继承中用到

> ordering 排序规则     `ordering = ['pub_date']` 给模型定义一个排序规则

> permissions 权限 我们可以给模型添加额外的权限像这样 `permissions = (("can_deliver_pizzas", "Can deliver pizzas"),)`注意他接收的是元组，元组的含义是这样的`(permission_code, human_readable_permission_name)`，每个模型都有默认的三种权限`('add', 'change', 'delete')`，前面的操作给他添加了一个额外的权限，在tamplate中可以使用`request.perm`来判断此用户是否具有此权限

> verbose name 给此模型一个自定义名字,verbose_name_plural模型的复数形式

Meta是非常有用的，[更多参考信息](https://docs.djangoproject.com/en/1.11/ref/models/options/)

##### Model methods

> 给你的模型定义方法，有一些已经有的方法，可以选择重写它们
例如`__str_()`和`get_absolute_url()`，当然还有`save()` `create()`等等 需要的时候去[这里看看](https://docs.djangoproject.com/en/1.11/ref/models/instances/)

    class Person(models.Model):
        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)

        # 用来定义object返回的str
        def __str__(self):
            return '%s %s' % (self.first_name, self.last_name)

        # 用来给某个object一个url 多用在RestFul中
        def get_absolute_url(self):
            return "/people/%i/" % self.id


    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def save(self, *args, **kwargs):
            if self.name == "Yoko Ono's blog":
                return # Yoko shall never have her own blog!
            else:
                super(Blog, self).save(*args, **kwargs) # Call the "real" save() method.

>>`super(Blog, self).save(*args, **kwargs)`不调用超类的`save`方法将无法保存数据

> 执行原生sql语句 两种方式
>> raw()

    class Person(models.Model):
        first_name = models.CharField(...)
        last_name = models.CharField(...)
        birth_date = models.DateField(...)

    # table_name如果没有在meta中指明则默认为app_model，假设我们的app名字为myapp就有了下面的myapp_person
    Person.objects.raw('SELECT id, first_name, last_name, birth_date FROM myapp_person')

当你看了文档发现还无法解决目前碰到的问题，那么只能出动原始sql了
> 原始sql

    from django.db import connection

    def my_custom_sql(self):
        with connection.cursor() as cursor:
            cursor.execute("UPDATE bar SET foo = 1 WHERE baz = %s", [self.baz])
            cursor.execute("SELECT foo FROM bar WHERE baz = %s", [self.baz])
            row = cursor.fetchone()

        return row

>> 这里就是python执行sql的最基础方法，先定义`connection`也就是数据库的连接信息，这里会直接从你的`settings.py`文件读取，然后就可以随心所欲写一些sql语句了

我这里只是一个简单的介绍，如果需要用到[请去看这里](https://docs.djangoproject.com/en/1.11/topics/db/sql/)

##### Model inheritance

> Abstract base classes 抽象基类(模型的父类)
>> 在一些模型中有很多公用的字段，可以抽出来作为一个基类，然后去继承他，定义的方式和普通的一样只是在`Meta`中要指明`abstract = True`，这样就不会再数据库中创建一个这个table

    from django.db import models

    class CommonInfo(models.Model):
        name = models.CharField(max_length=100)
        age = models.PositiveIntegerField()

        """
        抽象模型不会创建实体表
        """
        class Meta:
            abstract = True

    class Student(CommonInfo):
        home_group = models.CharField(max_length=5)

如果有`Meta`也要继承就像下面这样

    from django.db import models

    class CommonInfo(models.Model):
        # ...
        class Meta:
            abstract = True
            ordering = ['name']

    class Student(CommonInfo):
        # ...
        """
        这里django会自动设置  abstract=False  防止这个模型也变成一个抽象类
        如果你想创建一个抽象类继承另一个抽象类
        记得重复声明abstract = True
        """
        class Meta(CommonInfo.Meta):
            db_table = 'student_info'

>> 有一点需要注意，如果在基类中的`ForeignKey`或`ManyToManyField`使用了`related_name` `related_query_name`，定义的时候有一个规范保证唯一性防止出数据重复的错误

    from django.db import models

    class Base(models.Model):
        """
        这两个属性值必须包含 %(class)s 和 %(app_label)s

        %(class)s  会被替换成当前field使用的class名的小写而不是这个基类的名字

        %(app_label)s  会被替换成当前使用的app名字小写

        注意这个改动是从Django 1.10版本开始的
        """
        m2m = models.ManyToManyField(
            OtherModel,
            related_name="%(app_label)s_%(class)s_related",
            related_query_name="%(app_label)s_%(class)ss",
        )

        class Meta:
            abstract = True

    class ChildA(Base):
        pass

    class ChildB(Base):
        pass

如果没有定义`related_name` `related_query_name`，那还和普通的一样使用`_set`来进行查询

> Multi-table inheritance
>> 实体表的继承，就是不写`abstract = True`，父类也是个实体表，类似于下面的关系

    from django.db import models

    class Place(models.Model):
        name = models.CharField(max_length=50)
        address = models.CharField(max_length=80)

    class Restaurant(Place):
        serves_hot_dogs = models.BooleanField(default=False)
        serves_pizza = models.BooleanField(default=False)

这里的`Meta`继承就跟前面的`abstract`有区别了，父类的`Meta`对子类是无效的，除了极少数的情况`ordering` `get_latest_by`这两个属性会从父类中继承，如果不想继承这两个属性就在子类的`Meta`中显式的声明


    class ChildModel(ParentModel):
        # ...
        # proxy models 也是这么继承的
        class Meta:
            # Remove parent's ordering effect
            ordering = []
            # Remove parent's get_latest_by effect
            get_latest_by = ""


> Proxy models
>> 当你想对model有其他的操作而不想更改已经定义好的model时，代理模型就有用处了。定义起来跟普通的一样只是要在`Meta`中声明`proxy = True`，如果想给model添加一个方法


    from django.db import models

    class Person(models.Model):
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=30)

    class MyPerson(Person):
        class Meta:
            proxy = True

        def do_something(self):
            # ...
            pass

    # 换一个排序方式
    class OrderedPerson(Person):
        class Meta:
            ordering = ["last_name"]
            proxy = True

操作`MyPerson`和操作`Person`一样只是多了个`do_something()`，`proxy models`里的`Meta`的继承方式跟普通的一样

>> 注意`proxy models`只能继承自非抽象类`non-abstract model class`

关于`proxy models`的[更多信息](https://docs.djangoproject.com/en/1.11/topics/db/models/#proxy-models)

> Multiple inheritance

>> 多重继承几个要点
- 只有第一个`Meta`有效，其他的都会被忽略
- 如果没有设置主键，会出现`id`字段的冲突，前面有说过如果不设置主键，django会自动设置一个自增长`id`主键，所以要自己设置主键或者设置一个共同的父类，并用`OneToOneField`声明


    class Article(models.Model):
        article_id = models.AutoField(primary_key=True)
        ...

    class Book(models.Model):
        book_id = models.AutoField(primary_key=True)
        ...

    class BookReview(Book, Article):
        pass

    # 共同的父类像这样
    class Piece(models.Model):
        pass

    class Article(Piece):
        article_piece = models.OneToOneField(Piece, on_delete=models.CASCADE, parent_link=True)
        ...

    class Book(Piece):
        book_piece = models.OneToOneField(Piece, on_delete=models.CASCADE, parent_link=True)
        ...

    class BookReview(Book, Article):
        pass

>> 通常情况下不怎么用到多重继承，大多数情况可能是做一个`mix-in`，添加一个特定的字段或者是method，django建议我们尽量把model设计的简单明了以提高工作效率

##### Making_queries
> 前面大致介绍了如何定义继承扩展一个`model`，但是如何操作 `增删查改` 可能是大多数人最关心的一部分内容

假设我们的model是这样定义的

    from django.db import models

    class Blog(models.Model):
        name = models.CharField(max_length=100)
        tagline = models.TextField()

        def __str__(self):
            return self.name

    class Author(models.Model):
        name = models.CharField(max_length=200)
        email = models.EmailField()

        def __str__(self):
            return self.name

    class Entry(models.Model):
        blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
        headline = models.CharField(max_length=255)
        body_text = models.TextField()
        pub_date = models.DateField()
        mod_date = models.DateField()
        authors = models.ManyToManyField(Author)
        n_comments = models.IntegerField()
        n_pingbacks = models.IntegerField()
        rating = models.IntegerField()

        def __str__(self):
            return self.headline

> Creating objects (创建/保存)
>> 两种方式，`save()`和`create()`，后者的便利之处在于创建和保存在一行就可以完成。如果使用自定义的主键，要注意主键的唯一性

    >>> from blog.models import Blog
    >>> b = Blog(name='Beatles Blog', tagline='All the latest Beatles news.')
    >>> b.save()
    >>> c = Blog.objects.create(name='create Blog', tagline='create and save in one step.')

>> `save()`方法调用的时候才会真正的执行数据库save操作，他还可以对现有数据进行更新


    >>> b5.name = 'New name'
    >>> b5.save()

>>> Saving ForeignKey and ManyToManyField fields

外键和普通的一样`save()`

    >>> from blog.models import Blog, Entry
    >>> entry = Entry.objects.get(pk=1)
    >>> cheese_blog = Blog.objects.get(name="Cheddar Talk")
    >>> entry.blog = cheese_blog
    >>> entry.save()

多对多关系有一点区别，在保存`create()`之后需要把关系带上`add()`就像下面这样


    >>> from blog.models import Author
    >>> joe = Author.objects.create(name="Joe")
    >>> entry.authors.add(joe)

如果有多个就把多个同时`add()`

    >>> john = Author.objects.create(name="John")
    >>> paul = Author.objects.create(name="Paul")
    >>> george = Author.objects.create(name="George")
    >>> ringo = Author.objects.create(name="Ringo")
    >>> entry.authors.add(john, paul, george, ringo)

> Retrieving objects (查询)
>> `Blog.objects.all()`这个是最简单的一个搜索语句，`Blog`是`model`名，`objects`是`manager`名，`manager`是每个模型都会有的一个管理器可以改名字也可以自定义[详细看这里](https://docs.djangoproject.com/en/1.11/topics/db/managers/#managers)，`all()`是一个`filter`返回全部。
>>> 使用`filter`

`filter(**kwargs)`

返回符合条件的`Queryset`

`exclude(**kwargs)`

返回不符合条件的`Queryset`，是`filter(**kwargs)`的补集

`get()`与上面这俩的不同之处在于他返回`一个` `object`，如果没有此`object`或者有两个及以上的匹配会报错的，而使用`filter()`或者`exclude()`仅仅会返回`[]`或者返回更多的匹配

>>>> Field lookups

基本格式 `field__lookuptype=value`，字段名+双下划线+查询类型


查询类型有很多这里有[全部的参考](https://docs.djangoproject.com/en/1.11/ref/models/querysets/#field-lookups)

举几个栗子

    __exact        精确等于 like 'aaa'
     __iexact    精确等于 忽略大小写 ilike 'aaa'
    __contains    包含 like '%aaa%'
    __icontains    包含 忽略大小写 ilike '%aaa%'，但是对于sqlite来说，contains的作用效果等同于icontains。
    __gt    大于
    __gte    大于等于
    __lt    小于
    __lte    小于等于
    __in     存在于一个list范围内
    __startswith   以...开头
    __istartswith   以...开头 忽略大小写
    __endswith     以...结尾
    __iendswith    以...结尾，忽略大小写
    __range    在...范围内

当你需要在`同一个表的两个字段`做比较那么`F`就有用了 `F`的文档[在这里](https://docs.djangoproject.com/en/1.11/ref/models/expressions/#django.db.models.F)

可以在`F`对象上做一般的数学计算 加减乘除幂运算，双下划线也可以用 ...表示省略


    >>> from django.db.models import F
    >>> from datetime import timedelta
    >>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))
    ...
    >>> Entry.objects.filter(n_comments__gt=F('n_pingbacks') * 2)
    ...
    >>> Entry.objects.filter(rating__lt=F('n_comments') + F('n_pingbacks'))
    ...
    >>> Entry.objects.filter(authors__name=F('blog__name'))
    ...
    >>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))
    ...

`pk`是django提供的`primary key`的简称，简单用法简介如下

    >>> Blog.objects.get(id__exact=14) # Explicit form
    >>> Blog.objects.get(id=14) # __exact is implied
    >>> Blog.objects.get(pk=14) # pk implies id__exact

    # Get blogs entries with id 1, 4 and 7
    >>> Blog.objects.filter(pk__in=[1,4,7])

    # Get all blog entries with id > 14
    >>> Blog.objects.filter(pk__gt=14)

    >>> Entry.objects.filter(blog__id__exact=3) # Explicit form
    >>> Entry.objects.filter(blog__id=3)        # __exact is implied
    >>> Entry.objects.filter(blog__pk=3)        # __pk implies __id__exact

>> Complex lookups with Q objects (多条件查询不要怕有`Q`出马)

普通的`filter`都是以`AND`组合的，有时想用`OR`这时候`Q`可以解决你的需求，一个最简单的`Q`对象像这样，有两个连接符`|`(or), `&`(and),`~`(not)

    from django.db.models import Q
    Q(question__startswith='What')

    # 多个Q对象
    Q(question__startswith='Who') | Q(question__startswith='What') & Q(question__endswith='me')

在`filter()` `exclude()` `get()`中，==组合使用时Q必须在kwargs之前定义==

    Poll.objects.get(
        Q(question__startswith='Who'),
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
    )
    #相当于SQL中的
    """
    SELECT * from polls WHERE question LIKE 'Who%'
        AND (pub_date = '2005-05-02' OR pub_date = '2005-05-06')
    """

    # 可以和普通的语句组合
    """
    组合时Q对象必须在kwargs之前定义
    """
    # VALID QUERY
    Poll.objects.get(
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6)),
        question__startswith='Who',
    )

    # INVALID QUERY
    Poll.objects.get(
        question__startswith='Who',
        Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
    )


>> Comparing objects (比较)

>>> 使用`==`作为比较符，原理是比较两个实例的主键`primary key`，与你定义的主键字段名例如`person_id`无关，永远只会比较`primary key`，下面是栗子


    >>> some_entry == other_entry
    >>> some_entry.id == other_entry.id


    >>> some_obj == other_obj
    >>> some_obj.name == other_obj.name


>> Delete
>>> returns the number of objects deleted and a dictionary with the number of deletions per object type

    >>> Entry.objects.filter(pub_date__year=2005).delete()
    (5, {'webapp.Entry': 5})

需要注意的是他不会直接给`manager`调用，这是为了安全起见防止误操作，如果想删除表中的全部内容

    Entry.objects.all().delete()

>> Updating multiple objects at once
>>> 前面提到的`save()`似乎已经可以满足更新的需求了，但是还有有些时候不是那么方便

比如更新某个条件下的全部数据

    # Update all the headlines with pub_date in 2007.
    Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')

在更新外键的时候需要指明归属

    >>> b = Blog.objects.get(pk=1)

    # Change every Entry so that it belongs to this Blog.
    >>> Entry.objects.all().update(blog=b)


有一点需要注意`update`只能作用在一张表中，你可以使用各种`filter`但是最终只能`update`到你所使用的model中


    >>> b = Blog.objects.get(pk=1)

    # Update all the headlines belonging to this Blog.
    >>> Entry.objects.select_related().filter(blog=b).update(headline='Everything is the same')

和`F`组合，自增1

    >>> Entry.objects.all().update(n_pingbacks=F('n_pingbacks') + 1)

但是这里的`F`有一个限制就是不能加外部关系

> you can’t introduce joins when you use F() objects in an update – you can only reference fields local to the model being updated

    # This will raise a FieldError
    >>> Entry.objects.update(headline=F('blog__name'))

>> Related objects

前面已经把基础的增删查改介绍完了，但是实际的应用会在各种关系之下

>>> Many-to-one relationships

`ForeignKey`与`_set`不得不说的故事，前面定义多个`Entry`对应一个`Blog`

假设要从一个`Entry`得到相关联的`Blog`，很简单`.`一下就有了

    >>> e = Entry.objects.get(id=2)
    >>> e.blog # Returns the related Blog object.

但是反过来呢，我们知道Django里是没有一对多关系的，只有多对一关系，如果想得到某一个`Blog`下的所有`Entry`需要一个帮手`_set`反向查找，比如说会在template中使用`_set`生成一个sidebar，下面是个栗子


    >>> b = Blog.objects.get(id=1)
    >>> b.entry_set.all() # Returns all Entry objects related to Blog.

    # b.entry_set is a Manager that returns QuerySets.
    >>> b.entry_set.filter(headline__contains='Lennon')
    >>> b.entry_set.count()

有一个Manager叫做`FOO_set`，FOO在使用时替换成`source model`名小写，`FOO_set`可以通过定义`related_name`来自定义

比如把`Entry`中`blog`改成这样`blog = ForeignKey(Blog, on_delete=models.CASCADE, related_name='entries')`，查询语句就像下面这样


    >>> b = Blog.objects.get(id=1)
    >>> b.entries.all() # Returns all Entry objects related to Blog.

    # b.entries is a Manager that returns QuerySets.
    >>> b.entries.filter(headline__contains='Lennon')
    >>> b.entries.count()

关于`ForeignKey`的`Additional methods to handle related objects`可以[看这里](https://docs.djangoproject.com/en/1.11/ref/models/relations/)

>>> Many-to-many relationships

`ManyToManyField`唯一和`ForeignKkey`在查询时的区别是`_set`前面使用的是==该字段的名称==而不是`source model`小写，可以回去看一下最开始[定义的表结构](#making_queries)

    e = Entry.objects.get(id=3)
    e.authors.all() # Returns all Author objects for this Entry.
    e.authors.count()
    e.authors.filter(name__contains='John')

    a = Author.objects.get(id=5)
    a.entry_set.all() # Returns all Entry objects for this Author.


>>> One-to-one relationships

一对一关系比较简单看个栗子

    class EntryDetail(models.Model):
        entry = models.OneToOneField(Entry, on_delete=models.CASCADE)
        details = models.TextField()

可以通过一个属性直接找到

    ed = EntryDetail.objects.get(id=2)
    ed.entry # Returns the related Entry object.

    """
    由于是一对一关系 返回的只是个object了
    如果不存在会  raise a DoesNotExist exception
    """
    e = Entry.objects.get(id=2)
    e.entrydetail # returns the related EntryDetail object

以上只是简单的介绍，如果想深入了解去看一下这个[Models 最详细目录](https://docs.djangoproject.com/en/1.11/ref/models/)

这篇文章只是Django的基础知识点，可以作为一个概览，让你知道Django做了什么工作，我们如何利用他做一些事情

还有很多东西没有涉及到例如Form Admin Security cache signal等等实用的部分

如果想系统的学习Django那么[官方文档](https://docs.djangoproject.com/en/1.11/)是最好的途径，阅读文档+源码是最好的学习方式


本文参考：[https://docs.djangoproject.com/en/1.11/](https://docs.djangoproject.com/en/1.11/)
