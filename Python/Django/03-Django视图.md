[toc]
# 1 概述

## 1.1 视图作用
视图接受web请求，并响应web请求；
        
## 1.2 视图本质
视图就是一个python中的函数；
        
## 1.3 响应的内容
```
网页
        重定向
        错误视图    400、404、500
        
JSON数据
```

## 1.4 响应的过程
```
graph LR
A[<b>用户在浏览器输入网址</b><br>www.class.com/grade/index.html<br><br><b><i>网址</i></b>]              --> B
B[<b>django获取网址信息</b><br>去除ip与端口<br>grade/index.html<br><br><b><i>虚拟路径与文件名</i></b>]  --> C
C[<b>url管理器</b><br>逐个匹配urlconf<br>记录视图函数名<br><br><b><i>视图函数名</i></b>]                --> D
D[<b>视图管理器</b><br>找到对应的视图去执行<br>返回结果给浏览器<br><br><b><i>响应的数据</i></b>]        --> A
```

# 2 url配置
## 2.1 配置流程
```
1. 指定根级url配置文件
        配置 settings.py 文件中的 ROOT_URLCONF
        ROOT_URLCONF = 'class.urls'
        django默认配置好了

2. urlpatterns
        一个url实例列表
        url对象：
                正则表达式
                视图名称
                名称
                
3. url匹配正则的注意事项
        如果想要从url中获取一个值，需要对正则加小括号
                url(r'^stupage/(\d)/$', views.stupage),
        匹配正则前方不需要加 / 
        正则前需要加 r 表示字符串不转义
```
## 2.2 引入其他url配置
```
在应用中创建 urls.py 配置文件，定义本应用的url配置，在 工程/urls.py 文件中使用include()方法

        from django.conf.urls import url, include
        from django.contrib import admin
        urlpatterns = [
            url(r'^', include('myapp.urls', namespace='myapp'))
        ]

匹配过程：
        工程/urls.py --> 应用/urls.py
```

## 2.3 URL的反向解析
namespace='myapp'<br>name='myapp'
```
情景：
        如果修改了urls.py 中的正则表达式，则对应的模板中的 链接也都要做对应修改，非常影响工作；

概述：
        如果在视图、模板中使用了硬编码链接，在url配置发生改变时，动态生成链接地址；
        
解决：
        在使用链接时，通过url配置的名称，动态生成url地址；
        
作用：
        使用url模板
```

# 3 视图函数
## 3.1 定义视图
```
本质：
        一个函数；
        
视图参数：
        一个HttpRequest的实例，里面包含来自浏览器的全部内容；
        通过正则表达式获取的参数；
        
位置：
        一般在views.py文件下定义；
```

## 3.2 错误视图
```
400视图     错误出现在客户的操作
500视图     在视图代码中出现错误（服务器代码）

404视图
        找不到网页（url匹配不成功）时返回
        在template下定义404.html
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <title>404</title>
                </head>
                <body>
                    <h1>页面丢失</h1>
                    <h2>{{ request_path }}</h2>
                </body>
                </html>
                
                #------------------------------------
                request_path        导致错误的网址
                
        配置settings.py
                DEBUG       如果为True，永远不会调用404.html
                ALLOWED_HOSTS = ['*']
```

# 4 HttpRequest对象
## 4.1 概述
```
服务器接受http请求后，会根据报文创建一个HttpRequest对象；
视图的第一个参数就是 HttpRequest 对象；
Django创建的对象，之后传递给视图；
```

## 4.2 属性
```
path
        请求的完整路径（不包括域名和端口）；
        
method
        表示请求的方式，常用的有GET、POST；
        
encoding
        表示浏览器提交的数据的编码方式；
        一般为utf-8；
        
GET
        类似字典的对象，包含了get请求的所有参数；
        
POST
        类似字典的对象，包含了post请求的所有参数；
        
FILES
        类似字典的对象，包含了所有上传的文件；
        
COOKIES
        字典，包含所有的cookie
        
session
        类似字典的对象，表示当前的会话；
```

## 4.3 方法
```
is_ajax()
        如果是通过XMLHttpRequest发起的，返回True；
```

## 4.4 QueryDict对象
```
request对象中的GET、POST都属于QueryDict对象；

方法：
        get()
                作用：根据键获取值
                只能获取一个值
                www.class.com/abc?a=1&b=2&c=3       有一个a
        
        getlist()
                将键的值以列表的形式返回
                可以获取多个值
                www.class.com/abc?a=1&a=2&c=3       有两个a
```

## 4.5 GET属性
```
获取浏览器传递给服务器的数据；

http://127.0.0.1:8000/get1?a=1&b=2&c=3
        def get1(request):
            a = request.GET.get('a')
            b = request.GET.get('b')
            c = request.GET.get('c')
            return HttpResponse(a + '|' + b  + '|'+ c)
            
http://127.0.0.1:8000/get2?a=1&a=2&c=3
        def get2(request):
            a = request.GET.getlist('a')
            a1 = a[0]
            a2 = a[1]
            c = request.GET.get('c')
            return HttpResponse(a1 + '|' + a2 + '|' + c)
```

## 4.6 POST属性
```
使用表单
        编辑一个表单模板 register.html：
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <title>注册</title>
                </head>
                <body>
                    <form action="register/" method="post">
                        姓名：
                        <input type="text" name="name" value="">
                        <hr>
                        性别：
                        <input type="radio" name="gender" value="1">男
                        <input type="radio" name="gender" value="10">女
                        <hr>
                        年龄：
                        <input type="text" name="age" value="">
                        <hr>
                        爱好：
                        <input type="checkbox" name="hobby" value="power">权利
                        <input type="checkbox" name="hobby" value="money">金钱
                        <input type="checkbox" name="hobby" value="boy">美男
                        <input type="checkbox" name="hobby" value="girl">美女
                         <hr>
                        <input type="submit", value="注册">
                    </form>
                </body>
                </html>
                
        编辑urls.py：
                url(r'^showregister/$', views.showregister),
                url(r'^showregister/register/$', views.register),
                
        编辑视图 views.py：
                def showregister(request):
                    return render(request, 'myapp/register.html')
                def register(request):
                    name = request.POST.get('name')
                    gender = request.POST.get('gender')
                    age = request.POST.get('age')
                    hobby = request.POST.getlist('hobby')
                    print(name)
                    print(gender)
                    print(age)
                    print(hobby)
                    return HttpResponse('resgister')
                    
关闭CSRF
        注释掉 settings.py 中的 MIDDLEWARE: 'django.middleware.csrf.CsrfViewMiddleware'
                MIDDLEWARE = [
                    'django.middleware.security.SecurityMiddleware',
                    'django.contrib.sessions.middleware.SessionMiddleware',
                    'django.middleware.common.CommonMiddleware',
                    # 'django.middleware.csrf.CsrfViewMiddleware',
                    'django.contrib.auth.middleware.AuthenticationMiddleware',
                    'django.contrib.messages.middleware.MessageMiddleware',
                    'django.middleware.clickjacking.XFrameOptionsMiddleware',
                ]
```

# 5 HttpResponse对象
## 5.1 概述
```
作用：
        给浏览器返回数据；

HttpRequest对象是由Django创建的，HttpResponse对象是有程序员创建的
```
## 5.2 返回用法
```
不调用模板，直接返回数据
        def index(request):
            return HttpResponse('Hahahahaha')

调用模板，使用 render() 方法：
        原型：
                render(request, templateName[, context])
        
        作用：
                结合数据和模板，返回完整的HTML页面；
                
        参数：
                request         请求对象体
                templateName    模板路径
                context         传递给需要渲染在模板的数据
                
        示例：
                def grade(request):
                    gradeList = Grade.objects.all()
                    return render(request, 'myapp/grade.html', {'grade': gradeList})
```

## 5.3 属性
```
content
        表示返回的内容的类型

character
        返回的数据的编码格式
        
status_code
        响应状态码
                200
                304
                404
                
content-tpye
        指定输出的MIME类型
        
栗子：
        url(r'^showresponse/$', views.showresponse),
        
        def showresponse(request):
            res = HttpResponse()
            res.content = b'nice'
            print(res.content)
            print(res.charset)
            print(res.status_code)
            print(res.content-type)
            return res
```


## 5.4 方法
```
init
        使用页面内容实例化HttpResponse对象

writr(content)
        以文件的形式写入
        
flush
        以文件的形式输出缓冲区
        
set_cookie(key, value, max_age=None, exprise=None)
        

delete_cookie(key)
        删除 cookie
        注意： 如果删除一个不存在的cookie，什么都不会发生；

栗子：
        url(r'^cookietest/$', views.cookietest)
        
        def cookietest(request):
            res = HttpResponse()
            # res.set_cookie("class", "good")
            cookie = request.COOKIES
            res.write("<h1>" + cookie['class'] + "</h1>")
            return res
```

## 5.5 子类 HttpResponseRedirect
```
功能：
        重定向，服务器端的跳转；

栗子：
        url(r'^redirect1/$', views.redirect1),
        url(r'^redirect2/$', views.redirect2),

        from django.http import HttpResponseRedirect
        def redirect1(request):
            print('我是重定向前的redirect1')
            return HttpResponseRedirect('/redirect2')
        
        def redirect2(request):
            return HttpResponse('我是重定向后的redirect2')
            
            
简写：  redirect(to)    to 推荐使用反向解析
        
        from django.shortcuts import redirect
        def redirect1(request):
            print('我是重定向前的redirect1')
            return redirect('/redirect2')
        
        def redirect2(request):
            return HttpResponse('我是重定向后的redirect2')
```

## 5.6 子类 JsonResponse
```
返回json数据，一般用于异步请求（ajax）

__init__(self, data)
        data        字典对象

注意：
        Content-type 类型为 application/json

```



# 6 状态保持 session
## 6.1 概述
```
HTTP协议是无状态的，每次请求都是一次新的请求，不记得以前的请求；

客户端与服务器端的一次通信就是一次会话；

实现状态保持，在客户端或者服务器端存储有关会话数据；

存储的方式：
        cookie      所有的数据都存储在客户端，不要存储敏感数据；
        session     所有的数据存储在服务端，在客户端用cookie存储session_id；

状态保持的目的：
        在一段时间内，跟踪请求者的状态，可以实现跨页面访问当前请求者的数据；
        
注意：
        不同的请求者之间不会共享这个数据；
```

## 6.2 启用session
```
settings.py中：
        INSTALLED_APPS
                'django.contrib.sessions',
        MIDDLEWARE
                'django.contrib.sessions.middleware.SessionMiddleware',

默认启用
```

## 6.3 使用session
```
启用session后，每个HttpRequest对象，都有一个session属性，就是一个类似字典的对象；

get(key, default=None)
        根据键获取session值
        
clear()
        清空所有的会话
        
flush()
        删除当前的会话并删除会话的cookie
        
栗子：
        欢迎页 --登录--> 登录页 --输入用户名--> 欢迎页，显示登录名
        欢迎页 --退出登录--> 清理session
                urls.py
                        url(r'^main/$', views.main),
                        url(r'^login/$', views.login),
                        url(r'^showmain/$', views.showmain),
                        url(r'^quit/$', views.quit),
                
                views.py
                        from django.shortcuts import redirect
                        def main(request):
                            # 取session
                            username = request.session.get('name', '游客')
                            return render(request, 'myapp/main.html', {"username": username})
                        
                        def login(request):
                            return render(request, 'myapp/login.html')
                        
                        def showmain(request):
                            username = request.POST.get('username')
                            # 存储session
                            request.session['name'] = username
                            return redirect('/main')
                            
                        from django.contrib.auth import logout
                        def quit(request):
                            # 清除session
                            # logout(request)
                            request.session.clear()
                            return redirect('/main')
                    
                main.html
                        <head>
                            <meta charset="UTF-8">
                            <title>我的</title>
                        </head>
                        <body>
                            <h1>欢迎：{{ username }}</h1>
                            <a href="/login/">登录</a>
                            <a href="/quit/">退出登录</a>
                            
                        </body>
                    
                login.html
                        <head>
                            <meta charset="UTF-8">
                            <title>登录</title>
                        </head>
                        <body>
                            <form action="/showmain/" method="post">
                                <input type="text" name="username"><br>
                                <input type="submit" value="登录">
                            </form>
                        </body>
```
## 6.4 设置过期时间
```
set_expiry(value)
        如果不设置，默认是两个星期后过期；
        
        整数：  
                value填10，则是10秒后过期；
                
                def showmain(request):
                    username = request.POST.get('username')
                    # 存储session
                    request.session['name'] = username
                    request.session.set_expiry(10)
                    return redirect('/main')
                    
        时间对象
        
        0
                关闭浏览器是失效
        
        None
                永不过期
```
## 6.5 存储session的位置
```
在 settings.py 中增加 SESSION_ENGINE 配置；

数据库
        默认存储在数据库中；
        SESSION_ENGINE = 'django.contrib.sessions.backends.db'
        
缓存
        只存出在本地内存中，如果丢失，不能找回；
        SESSION_ENGINE = 'django.contrib.sessions.backends.cached'


数据库和缓存
        优先从本地缓存中读取，读取不到，再去数据库中获取；
        SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
        
```
## 6.6 使用redis缓存session
```
pip install django-redis-session

配置redis：
        在settings.py追加：
                SESSION_ENGINE = 'redis_sessions.session'
                SESSION_REDIS_HOST = 'localhost'
                SESSION_REDIS_PORT = 6379
                SESSION_REDIS_DB = 0
                SESSION_REDIS_PASSWORD = 'admin'
                SESSION_REDIS_PREFIX = 'session'
                
                
                
                
                
```
## 6.7
```

```
