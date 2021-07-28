[toc]
# 1 静态文件
```
定义：
        css、js、图片、json文件、字体文件等

存储位置：
        project/static/项目名/{css,js,img,json,font,other}
        
配置settings.py
        STATIC_URL = '/static/'
        STATICFILES_DIRS = [
            os.path.join(BASE_DIR, 'static'),
        ]
        
栗子：
        目录结构：
            Project
             └─static
                └─myApp
                   ├─css
                   ├─font
                   ├─img
                   ├─js
                   ├─json
                   └─other
        index.html
                <head>
                    <meta charset="UTF-8">
                    <title>index</title>
                    <link rel="stylesheet" type="text/css" href="/static/myApp/css/style.css">
                </head>
                    <script type="text/javascript" src="/static/myApp/js/lance.js"></script>
                <body>
                    <h1>lance is a good man</h1>
                    <img src="/static/myApp/img/a.png" alt="">
                </body>
                
        style.css
                h1{color: red;}
        
        lance.js
                console.log('good man')
                

普通文件动态加载static目录：（有错误 先别用）
        index.html
                {% load static from staticfiles %}
                <!DOCTYPE html>
                ...
                <img src="{% static 'myApp/img/a.png' %}" alt="">
                
        注意：
                这时的文件的链接路径 已经不需要从static开始去写了；
                settings.py中的 STATICFILES_DIRS 配置项已经无效；
                
                
```

# 2 中间件
## 2.1 概述
```
一个轻量级、底层的插件，可以介入Django的请求响应；
```

## 2.2 方法
```
__init__
        不需要传参数，服务器响应第一个请求的时候调用；用于确定是否启用该中间件；

process_request(self, request)
        在分配url匹配视图之前，每个请求上都会调用；返回 None 或者 HttpResponse 对象；

process_view(self, request, view_func, views_args, view_kwargs)
        调用视图之前执行，每个请求都会调用；返回 None 或者 HttpResponse 对象；

process_template(self, request, response)
        在视图执行完成后调用，每个请求都会调用；返回 None 或者 HttpResponse 对象；
        使用 render()
        
process_response(self, request, response)
        在执行模板之后调用，每个请求都会调用；返回 HttpResponse 对象；

process_exception(self, request, exception)
        当视图抛出异常时调用；返回 HttpResponse 对象；

```

## 2.3 自定义中间件
```
目录结构：
        project/middleware/myApp/
        
创建中间件：
        myMiddler.py
                from django.utils.deprecation import MiddlewareMixin
                class MyMiddler(MiddlewareMixin):
                    def process_request(self, request):
                        print("GET参数为：", request.GET.get("a"))

使用中间件：
        配置settings.py，在 MIDDLEWARE 中添加：
                'middleware.myApp.myMiddleware.MyMiddle',

验证：
        刷新界面，可以看到后台，有打印信息；
```

# 3 上传图片
## 3.1 概述
```
文件上传时，文件数据存储在request.FILES属性中；

注意：
        form表单要上传文件需要加 enctype="multipart/form-data" ；
        上传文件必须是POST请求；
```

## 3.2 存储配置

```
存储路径
        project/static/upfile

配置 settings.py：
        MEDIA_ROOT = os.path.join(BASE_DIR, 'static/upfile')
```


## 3.3 示例
```
upfile.html
        <form action="/savefile/" method="post" enctype="multipart/form-data">
            {% csrf_token %}
            <input type="file" name="file">
            <input type="submit" value="上传">
        </form>
        
views.py
        def upfile(request):
            return render(request, 'myApp/upfile.html')
        
        import os
        from django.conf import settings
        def savefile(request):
            if request.method == "POST":
                f = request.FILES["file"]
                # 文件在服务器端的路径
                filePath = os.path.join(settings.MEDIA_ROOT, f.name)
                with open(filePath, 'wb') as fp:
                    for info in f.chunks():     # 分段读取文件内容
                        fp.write(info)
                        fp.flush()
                return HttpResponse('上传成功')
            else:
                return HttpResponse("上传失败")
                
urls.py
        url(r'^upfile/$', views.upfile),
        url(r'^savefile/$', views.savefile),
```

# 4 分页
```
from django.core.paginator import Paginator
```

## 4.1 Paginator对象
```
创建对象
        格式：
                Paginator(列表, 整数)       # (所有的数据,每页几个)
        
        返回值：
                返回分页对象

属性
        count       对象总数
        
        num_pages   页面总数
        
        page_range  
                页码列表
                [1,2,3,4,5]
                页码从1开始

方法
        page(num)
                获得一个Page对象，如果提供的页码不存在，会抛出一个 'InvalidPage' 异常

异常
        InvalidPage
                当向 page() 传递一个无效的页码时抛出；
                
        PageNotAnInteger
                当向 page() 传递一个不是整数的参数时抛出；

        EmptyPage
                当向 page() 传递一个有效，但是该页面没有数据时抛出；
```

## 4.2 Page对象
```
创建对象
        Paginator对象的page方法返回得到Pafe对象；
        不需要手动创建；

属性
        object_list
                当前页上的所有的数据（对象）列表；
                
        number
                当前页的页码值；
        
        paginator
                当前page对象关联的paginator对象；
                
方法
        has_next()
                判断是否有下一页，如果有返回True；
                
        has_previous()
                判断是否有上一页，如果有返回True；
                
        has_other_pages()
                判断是否有上一页或者下一页，如果有返回True；
                
        next_page_number()
                返回下一页的页码，如故下一页不存在，抛出InvalidPage异常；
                
        previous_page_number()
                返回上一页的页码，如故上一页不存在，抛出InvalidPage异常；
                
        len()
                返回当前页的数据个数；
```

## 4.3 Paginator对象与Page对象的关系
```
Paginator对象 把 Student.object.all() 返回的列表分割；
使用 Paginator对象的 page方法，生成对应列表段的 Page对象；
每个Page对象，对应列表段中的内容；
```

## 4.4 示例：分页显示学生信息
```
studentpage.html
    <ul><li>{{ pagenum }}</li></ul>
    <ul>
        {% for stu in student %}
            <li>{{ stu.id }} | {{ stu.sname }} | {{ stu.sgrade }}</li>
        {% endfor %}
    </ul>
    <ul>
        {% for index in student.paginator.page_range %}
        {# 调用page对象的paginator方法，到达paginator类；#}
        {# 再使用paginator类的page_range方法；#}
            <li><a href="/studentpage/{{ index }}/">{{ index }}</a></li>
        {% endfor %}
    </ul>
    
views.py
        from django.core.paginator import Paginator
        def studentpage(request, pageid):
            # 所有学生的列表
            allList = Student.stuObj.all()
            paginator = Paginator(allList, 3)
            page = paginator.page(pageid)
            pagenum = page.number
            return render(request, 'myapp/studentpage.html', {"student": page, "pagenum": pagenum})

urls.py
        url(r'^studentpage/(\d+)/$', views.studentpage),
```

# 5 ajax
```
需要动态生成，请求json数据

栗子：
        views.py
                from django.http import JsonResponse
                def ajaxstudet(request):
                    stus = Student.stuObj.all()
                    list = []
                    for stu in stus:
                        list.append([stu.id, stu.sname, stu.sgrade])
                    return JsonResponse({"data": list})
```


# 6 富文本
```
pip install django-tinymce

在站点中使用
        配置settings.py文件
                在INSTALLED_APPS中添加：'tinymce',
                在末尾追加：
                        # 富文本
                        TINYMCE_DEFAULT_CONFIG = {
                            'theme': 'advanced',
                            'width': 600,
                            'height': 400,
                        }
                        
        创建一个模型类（myApp/models.py）：
                from tinymce.models import HTMLField
                class Text(models.Model):
                    str = HTMLField()
                    
        配置站点（myApp/admin.py）：
                from .models import Text
                admin.site.register(Text)
                

在自定义视图中使用
        urls.py
                url(r'^edit/$', views.edit),
                
        views.py
                def edit(request):
                    return render(request, 'myApp/edit.html')
                    
        edit.html
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <title>富文本</title>
                    <script type="text/javascript" src="/static/tiny_mce/tiny_mce.js"></script>
                    <script type="text/javascript">
                        tinyMCE.init({
                            'mode': 'textareas',
                            'theme': 'advanced',
                            'width': 600,
                            'height': 400,
                        })
                    </script>
                </head>
                <body>
                <form action="">
                    <textarea name="str">lance is good man</textarea>
                    <input type="submit" value="提交">
                </form>
                </body>
                </html>
```

# 7 celery
> http://docs.jinkan.org/docs/celery/

```
问题：
        1. 用户发起request，并且要等待response返回；但是在视图中有一些耗时的操作，导致用户可能会等待很长时间才能接受response，这样用户体验很差；
        
        2. 网站每隔一段时间要同步数据，但是http请求是需要触发的；


celery解决：
        问题1： 将耗时的操作，放到celery中执行；
        问题2： 使用celery定时执行；


celery组件：
        任务tasl        本质是一个Python函数，将耗时操作封装成一个函数；
        队列queue       将要执行的任务，放到队列里；
        工人woeker      负责执行队列中的任务；
        代理broker       负责调度，在部署环境中使用redies;


安装：
        pip install celery
        pip install celery-with-redis
        pip install django-celery
       
        
配置settings.py:
        INSTALLED_APPS
                'djcelery',
                
        追加
                import djcelery
                djcelery.setup_loader()     #初始化
                BROKER_URL = 'redis://:user@IP:6379/0'
                CELERY_IMPORTS = ('myApp.task',)
            
               
创建task.py
        project/myApp/task.py


迁移，生成celery需要的数据库表:
        python manage.py migrate


在 /project/project 下创建 celery.py:
        from __future__ import absolute_import
        import os
        from celery import Celery
        from django.conf import settings
        
        os.environ.setdefault( 'DJANGO_SETTINGS_MODULES', 'whthas_home.settings')
        app = Celery('portal')
        
        app.config_from_onject('django.conf:settings')
        app.authodiscover_tasks(lambda: settings.INSTALLED_APPS)
        
        @app.task(bind=True)
        def debug_task(self):
        print('Request: {0!r}'.format(self.request))


在project/project/__init__.py中添加：
    from .celery import import app as celery_app
    









```





