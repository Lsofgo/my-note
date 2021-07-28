[toc]
# 1 


# 2 定义模板
## 2.1 变量
```
视图传递给模板的数据；

传递的变量要遵守标识符规则；

语法：
        {{ 变量 }}
        num = {{ num }}

注意：
        如果使用的变量不存在，则插入的是空字符串；

在模板中使用点语法（对象.属性）：
        字典查询
        属性或方法
        数字索引
        
在模板中调用对象的方法：
        注意：  不能传递参数
```

## 2.2 标签
```
语法：
        {% tag %}


作用：
        1. 在输出中创建文本
        2. 控制逻辑和循环
        
if
        格式：
                {% if 表达式1 %}
                    语句
                {% endif %}
            # --------------------------
                {% if 表达式1 %}
                    语句
                {% else %}
                    语句
                {% endif %}
            # --------------------------
                {% if 表达式1 %}
                    语句
                {% elif 表达式2 %}
                    语句
                ......
                {% else %}
                    语句
                {% endif %}
                
        栗子：
                test.html
                        <body>
                            {% if num %}
                            <h1>Python is Best;{{ num }}</h1>
                            {% endif %}
                        </body>
                        
                views.py
                        def templateTest(request):
                            num = 10
                            return render(request, 'myapp/test.html', {'num':num})
                            
                urls.py
                        url(r'^templateTest', views.templateTest),

for
        格式：
                {% for 变量 in 列表 %}
                    语句
                {% endfor %}
                # ----------------------------------------------------------
                {% for 变量 in 列表 %}
                    语句1
                {% empty %}
                    语句2
                {% endfor %}
                        注意： 列表为空或者不存在时，执行语句2
                # ----------------------------------------------------------      
                {{ forloop.counter }}   在for内使用，表示当前第几次循环；
                
        栗子：
                test.html
                        <body>
                            <ul>
                                {% for stu in stulist %}
                                    <li>{{ forloop.counter }} | {{ stu.sname }} | {{ stu.sgrade }}</li>
                                {% empty %}
                                    <h1>这里没有学生</h1>
                                {% endfor %}
                            </ul>
                        </body>
                        
                views.py
                        def templateTest2(request):
                            stulist = Student.stuObj2.all()
                            return render(request, 'myapp/test.html', {'stulist':stulist})
                            
                urls.py
                        url(r'^templateTest2/$', views.templateTest2),

comment
        作用：
                多行注释
                
        格式：
                {% comment %}
                    注释的内容
                {% endcomment %}

ifequal/ifnotequal
        作用：
                判断相等/判断不相等
                
        格式：
                {% ifequal 值1 值2 %}
                    语句
                {% endifequal %}
                
                当 值1 == 值2，则执行 语句；否则不执行语句；

include
        作用：
                加载模板，并以标签内的参数渲染
        
        格式：
                {% include '模板目录' 参数1 参数2 %}

url
        作用：
                反向解析
                
        格式：
                {% url'namespace:name' 参数1 参数2 %}

csrf_token
        作用：
                用于跨站请求伪造保护
                
        格式：
                {% csrf_token %}

block/extends
        作用：
                用于模板继承

autoescape
        作用：
                用于html转义


```

## 2.3 过滤器
```
        作用：
                在变量被显示前修改它
                
        语法：
                {{ 变量 | 过滤器 }}
        
        栗子：
                test.html
                        <body>
                            {{ python|upper }}
                        </body>
                        
                views.py
                        def templateTest(request):
                            num = 10
                            return render(request, 'myapp/test.html', {'num': num, 'python': 'python is best.'})
                            
        过滤器：
                upper
                        {{ python|upper }}
                lower
                过滤器可以传递参数，参数用引号引起来；
                        join
                                格式： {{ 列表|join:'#' }}
                                栗子：
                                        test.html
                                                {{ list|join:'#' }}
                                        
                                        views.py
                                                templateTest(request):
                                                return render(request, 'myapp/test.html', {'list': ['Abc', 'qqq', 'qwer']})
                                                
                        default
                                作用：如果一个变量没有被提供，或者值为Flase、空，可以使用默认值 
                                格式： {{ 变量|default:'good' }}
                                
                        date
                                作用： 根据给定格式转换日期为字符串
                                格式： {{ date变量|date:'y-m-d'}}
                                
                        escape
                                作用： HTML转义
                                
                        safe
                                作用： HTML转义
                                格式： {{ 变量|safe }}                                
                                         
                        加减乘除
                                加：num = {{ num|add:10 }}
                                减：num = {{ num|add:-10 }}
                                乘：{% widthratio num 1 5 %}  # num/1*5
                                除：{% widthratio num 2 1 %}  # num/2*1
                                
                        divisibleby
                                num|divisibleby:2
                                        num/2 能除尽，则返回 True；
```


## 2.4 注释
```
单行注释
        <# 注释的内容 #>
多行注释
        <% connment %>
        <% endcomment %>
```

## 2.5 反向解析
```
通过定义在 urls.py 文件中的 namespace 和 name 参数，在模板中拼接出href参数，避免由于urls.py中的正则修改而大量的修改模板参数；

使用方法：
        自动拼接出 127.0.0.1:8000/good/1/:
        
        project/urls.py
                url(r'^', include('myApp.urls', namespace='myApp')),
        
        myApp/urls.py
                url(r'^good/(\d+)/$', views.good, name='good'),
                
        views.py
                def good(request, num):
                    return render(request, 'myApp/good.html', {'num': num})
                        
        index.html
                <h1><a href="/good/123">链接123</a></h1><br>
                <h1><a href="{% url 'myApp:good' 1 %}">链接</a></h1>
                
        good.html
                <h1>good--{{ num }}</h1>
```

## 2.6 模板继承
```
作用：
        模板继承，可以减少页面内容的重复定义，实现页面的重用；
        
block标签：
        在父模板中预留区域；
        在子模板去填充父模板的预留区域；
        语法：
                {% block 标签名 %}
                    ......
                {% endblock 标签名 %}


extends标签
        继承模板，需要卸载模板文件的第一行；
        语法：
                {% extends 父模板路径 %}
        
栗子：
        定义父模板：
                base.html
                    <div id="header">header</div>
                    <div id="main">
                        {% block main%}
                
                        {% endblock %}
                    </div>
                    <div id="footer">footer</div>
                
        定义子模板：
                main.html
                    {% extends 'myApp/base.html' %}
                    {% block main %}
                        <h1>lance is a good man</h1>
                    {% endblock %}
                    
        urls.py
                url(r'^main/$', views.main),
                
        views.py
                def main(request):
                    return render(request, 'myApp/main.html')
```

## 2.7 html转义
```
问题：
        在 views.py 中传递 html 代码，在模板中，会将接受到的代码当字符串渲染；
                good.html
                    {{ code }}
                
                views.py
                    def good(request):
                        return render(request, 'myApp/good.html', {'code':'<h1>lance is a good man</h1>'})

解决：
        方法一：safe过滤器
            {{ code|safe }}
            
        方法二：autoescap标签
            {%  autoescape off %}       # 设置为on，则会当成字符串处理；
            {{ code }}{{ code }}
            {% endautoescape %}
        
```

## 2.8 CSRF
```
跨站请求伪造
        某些恶意网站包含链接、表单、按钮、DNS，会利用登录用户在浏览器中的认证，攻击服务；
        
防止CSRF
        在 settings.py 的 MIDDLEWARE 增加 'django.middleware.csrf.CsrfViewMiddleware'；
        
        在表单中加入 csrf_token 标签，防止自己的表单被拦截；
                <form action="/showinfo/" method="post">
                {% csrf_token %}
                姓名：<input type="text" name="username">
                <hr>
                密码：<input type="password" name="password">
                <hr>
                <input type="submit" value="登录">
</form>
```

## 2.9 验证码
```
作用：
        在用户注、登录页面时候，防止暴力请求，减轻服务器压力；
        防止csrf的一种方式；
        
栗子：
        views.py
                # 绘制验证码
                def verifycode(request):
                    # 引入绘图模块
                    from PIL import Image, ImageDraw, ImageFont
                    # 引入随机函数模块
                    import random
                    # 引入字符串模块
                    import string
                
                    # 设置画面背景色，从20到100中随机生成RGB颜色代码
                    bgcolor = (random.randrange(20,100), random.randrange(20,100), random.randrange(20,100))
                    # 设置画面宽、高
                    width = 100
                    height = 50
                
                    # 创建画面对象
                    im = Image.new('RGB', (width, height), bgcolor)
                    # 创建画笔对象
                    draw = ImageDraw.Draw(im)
                    # 调用画笔的point函数绘制噪点
                    for i in range(0,100):
                        xy = (random.randrange(0, width), random.randrange(0, height))
                        fill = (random.randrange(0, 255), 255, random.randrange(0, 255))
                        draw.point(xy, fill=fill)
                    # 定义验证码备选值
                    str = string.digits + string.ascii_letters
                    # 选取4个值作为验证码
                    rand_str = ''
                    for i in range(0,4):
                        rand_str += str[random.randrange(0, len(str))]
                    # 构造字典体对象
                    font = ImageFont.truetype(r'./arial.ttf', 40)
                
                    # 构造字体颜色
                    fontcolor = (255, random.randrange(0, 255), random.randrange(0, 255))
                    # 绘制4个字体
                    draw.text((5, 2), rand_str[0], font=font, fill=fontcolor)
                    draw.text((25, 2), rand_str[1], font=font, fill=fontcolor)
                    draw.text((50, 2), rand_str[2], font=font, fill=fontcolor)
                    draw.text((75, 2), rand_str[3], font=font, fill=fontcolor)
                    # 释放画笔
                    del draw
                    # 存入session，用于做进一步验证
                    request.session['verify'] = rand_str
                    # 内存文件操作
                    import io
                    buf = io.BytesIO()
                    # 将图片文件保存在内存中，文件类型png
                    im.save(buf, 'png')
                    return HttpResponse(buf.getvalue(), 'image/png')
                
                # 登录页面
                def verifycodefile(request):
                    return render(request, 'myApp/verifycodefile.html')
                
                # 登录验证
                from django.shortcuts import redirect
                def verifycodecheck(request):
                    code1 = request.POST.get('verifycode').upper()
                    code2 = request.session.get('verify').upper()
                    print(code1)
                    print(code2)
                    if code1 == code2:
                        return  render(request, 'myApp/success.html')
                    else:   # 登录失败返回登录页面
                        return  redirect('/verifycodefile/')
                        
        #-----------------------------------------------------------------------------------
        urls.py
                # 验证码图片的url
                url(r'^verifycode/$', views.verifycode),
                # 登录界面url
                url(r'^verifycodefile/$', views.verifycodefile),
                # 登录验证
                url(r'^verifycodecheck/$', views.verifycodecheck),
                
        #-----------------------------------------------------------------------------------
        verifycodefile.html
                <form action="/verifycodecheck/" method="post">
                    {% csrf_token %}
                    姓名：<input type="text" name="username">
                    <hr>
                    密码：<input type="password" name="password">
                    <hr>
                    验证码：<input type="text" name="verifycode">
                    <img src="/verifycode/" alt="">
                    <input type="submit" value="登录">
                </form>
```
