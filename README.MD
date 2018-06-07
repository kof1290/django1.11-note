# Django基础知识点整理

`django1.11`

> ## 目录结构如下

- Url
    - project url
    - app url
        - RegEx
        - args
        - kwargs

- View
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
- Template
    - Variables
    - Template inheritance
    - Custom tag and filter libraries
        - tags
        - apptags
        - filters
    - context processors
    - static
- Model
    - define and use
    - Fields
        - relationship
            - OneToOne
            - ManyToOne
            - ManyToMany
    - Meta options
    - Model methods
    - Model inheritance
    - Making queries



这篇文章只是Django的基础知识点，可以作为一个概览，让你知道Django做了什么工作，我们如何利用他做一些事情

还有很多东西没有涉及到例如Form Admin Security cache signal等等实用的部分

如果想系统的学习Django那么[官方文档](https://docs.djangoproject.com/en/1.11/)是最好的途径，阅读文档+源码是最好的学习方式


本文参考：[https://docs.djangoproject.com/en/1.11/](https://docs.djangoproject.com/en/1.11/)
