[toc]
# 1 安装Django
## 1.1 Django版本与Python版本对应关系
    Django      | Python
    ------      | ------
    1.8         | 2.7, 3.2 (until the end of 2016), 3.3, 3.4, 3.5
    1.9, 1.10   | 2.7, 3.4, 3.5
    1.11        | 2.7, 3.4, 3.5, 3.6
    2.0         | 3.4, 3.5, 3.6
    2.1         | 3.5, 3.6, 3.7

## 1.2 安装Django

安装<br>
pip install django==1.11.4<br>
<br>
卸载<br>
pip uninstall django<br>

## 1.3 安装验证
```
python
>>> import django
>>> django.get_version()
```

# 2 创建项目

```
在一个合适的位置创建目录

使用终端进入上一步创建的目录下

执行： django-admin startproject [project_name]

目录层级说明：
    project
        manage.py           一个命令行工具，可以让我我们使用多种方式对Django项目进行操作
        
        project目录
            __init__.py     一个空文件，告诉Python这个目录应该被看做一个Python包
            settings.py     项目的配置文件
            urls.py         项目的URL声明，匹配view
            wsgi.py         项目与WSGI兼容的Web服务器入口
```

# 3 基本操作
## 3.1 设计表结构
```
学生库
        班级表结构
                表名    grade
                字段
                        班级名称    gname
                        成立时间    gdate
                        女生总数    ggirlnum
                        男生总数    gboynum
                        是否删除    isDelete
        学生表结构
                表名    students
                字段
                        学生姓名    sname
                        学生性别    sgender
                        学生年龄    sage
                        学生简介    scontend
                        所属班级    sgrade
                        是否删除    isDelete
```

## 3.2 配置数据库
```
注意：Django 默认使用sqlite3

在settings.py文件中，通过DATABASES进行数据库配置

配置MySQL
        python3.x 安装的是PyMySQL； python2.x 安装的是mysql-python
        
        在__init__.py写入两行代码
            import pymysql
            pymysql.install_as_MySQLdb()
        
        配置 settings.py
            DATABASES = {
                'default': {
                    'ENGINE': 'django.db.backends.mysql',
                    'NAME': 'class',
                    'USER': 'root',
                    'PASSWORD': 'admin',
                    'HOST': '192.168.23.128',
                    'PORT': '3306',
                }
            }
```

## 3.3 创建应用
在每个项目中可以创建多个应用，每个应用进行一种业务处理；
```
打开终端，进入项目目录下；

执行： python manage.py startapp myapp

myapp 目录说明：
        admin.py        站点配置
        models.py       模型
        views.py        视图
```

## 3.4 激活应用
```
在 settings.py 文件中，将myapp应用加入到 INSTALLED_APPS 中；

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'myapp'
]

```
    
## 3.5 定义模型
```
概述： 有一个数据表，就对应一个模型；

在 models.py 文件中定义模型：
        引入 from django.db import models
        
        模型要继承 models.Model 类
        
                class Grades(models.Model):
                    gname = models.CharField(max_length=20)
                    gdate = models.DateTimeField()
                    ggirlnum = models.IntegerField()
                    gboynum = models.IntegerField()
                    isDelete = models.BooleanField(default=False)
                    
                class Students(models.Model):
                    sname = models.CharField(max_length=20)
                    sgender = models.BooleanField(default=True)
                    sage = models.IntegerField()
                    scontend = models.TextField()
                    isDelete = models.BooleanField(default=False)
                    # 关联外键
                    sgrade = models.ForeignKey('Crades')

说明：
        不需要定义主键，在生成表时自动添加，并且值为自动增加；
        一个表对应一个模型类
        一个类属性对应标的一个字段
```

## 3.6 在数据库中生成数据表
```
生成迁移文件
        执行： python manage.py makemigrations <appname>
        迁移文件在：myapp/migrations/0001_inital.py
        此时数据库中还没有生成数据表

执行迁移：
        执行： python manage.py  migrate <appname>
        相当于执行 sql 语句创建数据表

备注：
        带上appname只会生成对应app的数据表；
        不带，则会生成全部的数据表，包括一些系统的默认表；
```

## 3.7 测试数据操作
```
进入python shell，执行： python manage.py shell

引入一些包
        from myapp.models import Grands, Students
        from django.utils import timezone
        from datatime import *
        
查询所有数据
        类名.objects.all()
        Grades.objects.all()
        
        重写 __str__方法，可以看到对象的值：
            def __str__(self):
                return '%s | %s | %s' % (self.gname, self.gdate, self.gnum)
        
添加数据
        本质： 创建一个模型类的对象实例；
            grade1 = Grades()
            grade1.gname = 'python'
            grade1.gdate = datetime(year=2018, month=2, day=14)
            grade1.ggirlnum=3
            grade1.gboynum=2
            grade1.save()
            
查看某个对象
        类名.objects.get(pk=2)      pk == primary_key
        
修改数据
        grade2 = Grades.objects.get(pk=2)
        grade2.ggirlnum = 20
        grade2.save()
        
删除数据
        grade2.delete()
        注意：是物理删除，数据库中的表里的数据被删除了；

关联对象
            grade1 = Grades.objects.get(pk=1)
            stu = Students()
            stu.sname = 'jane'
            stu.sgender=False
            stu.sage=18
            stu.scontend= "i am jane."
            stu.sgrade = grade1
            stu.save()
        
        获取关联对象的集合
                需求：获取 python 班所有学生
                        对象名.关联的类名小写_set.all()
                            grade1.students_set.all()
        
            
                需求：给 python 班创建allen
                            stu3 = grade1.students_set.create(sname=u'allen', sgender=True, sage=30, scontend='i am allen.')
                        注意：不用调用save方法
```
## 3.8 启动服务器
```
格式：
        执行：python manage.py runserver ip:port
        ip 可以不写，不写代表localhost
        port默认8000
        执行：python manage.py runserver

说明：
        这是纯python写的轻量级web服务器，只在测试开发时使用；

```

## 3.9 admin站点管理
```
概述：
        内容发布：
                负责添加、修改、删除内容
                公告访问

配置admin应用：
        在 setting.py 文件的 INSTALLED_APPS 中添加 django.contrib.admin


创建管理员用户：
        执行：python manage.py createsuperuser
        一次输入 用户名、邮箱、密码(8位，非字典密码)
        访问：127.0.0.1:8000/admin


汉化：
        修改 settings.py
                LANGUAGE_CODE = 'zh-Hans'
                TIME_ZONE = 'Asia/Shanghai'


管理数据表：
        修改 admin.py：
                from .models import Grades, Students
                # 注册
                admin.site.register(Grades)
                admin.site.register(Students)
            
            
        自定义管理页面：(上面的注册抛掉)
                class GradesAdmin(admin.ModelAdmin):
                
                # 列表页属性
                    list_display = ['pk', 'gname', 'gdate', 'gbnum', 'ggnum']       # 显示表的字段(pk 是主键，也可以用字段名id)
                    list_filter = ['gname']             # 右侧添加过滤字段栏，使用字段以及字段值来过滤数据
                    search_fields = ['gdate']           # 增加搜索框，定义哪些字段的值能够搜索
                    list_per_page = 5                   # 分页，每页显示的数据数量
            
                # 添加、修改页的属性 (注意 fields 和 fieldsets 不能同时使用)
                    fields = ['gname', 'gbnum', 'ggnum', 'gdate', 'isDelete']         # 属性的先后顺序：修改表单的显示顺序
                    
                    fieldsets = [                                           # 给属性分组
                        ('Base', {"fields": ['sname', 'sgender', 'sage']}),
                        ('Other', {"fields": ['sgrade', 'scontend'， 'isDelete']})
                    ]
                    
                admin.site.register(Grades, GradesAdmin)
            
        
        关联对象：
                需求：在创建一个班级时，可以直接添加几个学生；
                        class StudentsInfo(admin.TabularInline):
                            model = Students        # 和 Students 模型关联
                            extra = 2               # 创建班级时，添加两个学生
                        # 注册
                        class GradesAdmin(admin.ModelAdmin):
                            inlines = [StudentsInfo]
                备注：
                        admin.TabularInline     把需要填写的表单，放在一行显示
                        admin.StackedInline     把需要填写的表单，逐行显示
                    
                    
        boole值显示问题：
                class StudentsAdmin(admin.ModelAdmin):
                    def gender(self):
                        if self.sgender:
                            return '男'
                        else:
                            return '女'
                # 设置页面列的名称
                    gender.short_description = '性别'
                    list_display = ['id', 'sname', 'sage', gender, 'scontend', 'sgrade']
                
        
        列表标题显示问题：
                age = lambda self: self.sage
                age.short_description = '年龄'
                list_display = ['id', 'sname', age, gender, 'scontend', 'sgrade']
        
            
        执行动作操作框位置：
            actions_on_bottom = True
            actions_on_top = False      # 默认是 True
        
        
        使用装饰器完成注册
            @ admin.register(Students)
            class StudentsAdmin(admin.ModelAdmin):
            ...
            ...
            # admin.site.register(Students, StudentsAdmin)

```


# 4 视图的基本使用
## 4.1 概述
在 Django 中，视图对web请求进行回应；<br>
视图就是一个 python 函数，在 views.py 中定义；<br>


## 4.2 定义视图
```
from django.shortcuts import render

# Create your views here.
from django.http import HttpResponse
def index(request):
    return HttpResponse('Python is Best!')
```

## 4.3 配置url
```
修改 class(工程) 目录下的 urls.py 文件
    from django.conf.urls import url,include
    from django.contrib import admin
    urlpatterns = [
        url(r'^admin/', admin.site.urls),
        url(r'^', include('myapp.urls'))
    ]


在myapp应用目录下创建一个 urls.py
    from django.conf.urls import url
    from . import views
    urlpatterns = [
        url(r'^$', views.index)
    ]

```
# 5 模板的基本使用
## 5.1 概述
模板是HTML页面，可以根据视图传递过来的数据进行填充；

## 5.2 创建模板
```
创建 templates 目录，在目录下创建对应模板目录 class/templates/myapp

```
## 5.3 配置模板路径
```
修改 settings.py 文件下 TEMPLATES, 把templates添加到DIRS中：
    TEMPLATES = [
        {
            'BACKEND': 'django.template.backends.django.DjangoTemplates',
            'DIRS': [os.path.join(BASE_DIR, 'templates')]，
            'APP_DIRS': True,
```

## 5.4 定义模板
```
在 class/templates/myapp 下创建 grades.py 和 students.py
        模板语法：
                {{输出值，可以是变量，也可以是对象.属性}}
                {%执行代码段%}
        
        
        需求：http://127.0.0.1:8000/grades
        
                编辑 grades.html 模板:
                        <h1>班级信息列表</h1>
                        <ul>
                            {% for grade in grages %}
                            <li>
                                <a href="#">{{ grade.gname }}</a>
                            </li>
                            {% endfor %}
                        </ul>
                    
                    
                定义视图：
                        from .models import Grades
                        def grades(requests):
                            # 去模板取数据
                            gradesList= Grades.objects.all()
                            # 将数据传递给模板，再渲染页面，将渲染好的页面返回给浏览器
                            return render(requests, 'myapp/grades.html', {'grades': gradesList})
                            #            request参数     模板路径      模板中接收的参数名: 传递的数据
                        
                配置url:
                        url(r'^grades/$', views.grades),
                    
                    
        需求：http://127.0.0.1:8000/students
                编辑 students.html 模板：
                        <h1>学生信息表</h1>
                        <ul>
                            {% for studentInfo in student %}
                            <li>
                                <a href="#">{{ studentInfo.id }}-{{ studentInfo.sname }}-{{ studentInfo.sage }}-{{ studentInfo.scontend }}</a>
                            </li>
                            {% endfor %}
                        </ul>
                    
                定义视图：
                        def student(request):
                            studentList = Student.objects.all()
                            return render(request, 'myapp/student.html', {'student': studentList})
                    
                配置url：
                        url(r'^student/$', views.student),
        
        
        需求：点击班级显示对应学生信息
                复用 students.html 模板
                
                修改 grades.html 让UAL传参：
                        <a href="{{ gradeInfo.id }}">{{ gradeInfo.gname }}</a>
                    
                定义视图：
                        def gradeStudent(request, num):
                            grade = Grade.objects.get(pk=num)
                            studentList = grade.student_set.all()
                            return render(request, 'myapp/student.html', {'student': studentList})
                            
                配置url：
                        url(r'^grade/(\d)$', views.gradeStudent),
                        
        
        需求：通过主页分别访问 grades students admin
                编辑 index.html 模板：
                        <h2><a href="{{ 'admin' }}">admin</a></h2>
                        <h2><a href="{{ 'grade' }}">Grade</a></h2>
                        <h2><a href="{{ 'student' }}">Student</a></h2>
                        
                修改视图：
                        def index(request):
                            return render(request, 'myapp/index.html')
                            
                配置url：
                        url(r'^$', views.index),
```


