[toc]
# 1 模型

Django对各种数据库都提供了很好的支持，Django为这些数据库提供了统一的调用API，可以根据不同的业务需求，选择不同的数据库；

## 1.1 开发流程
```
配置数据库:
        python3.x 安装的是PyMySQL； python2.x 安装的是mysql-python
        
        在工程目录下的__init__.py写入两行代码
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


定义模型类
        一个模型类都在数据库中队形一张数据表；

生成迁移文件

执行迁移

使用模型类进行增、删、改、查
```
## 1.2 ORM
### 1.2.1 概述
>对象-关系-映射

### 1.2.2 任务
- 根据对象类型生成表结构；
- 将对象、列表的操作转换为SQL语句；
- 将SQL语句查询到的结果转换为对象、列表；

### 1.2.3 优点
- 极大的减轻了开发人员的工作量，不需要面对因数据库的改变而修改代码；

### 1.2.4 图解
```
graph TD
    A[Django] 
    B[ORM]
    C1[MySQL]
    C2[sqlite]
    C3[Oracle]
```

## 1.3 定义模型

### 1.3.1 模型、属性、表、字段间的关系
- 一个模型类在数据库中对应一张表；
- 在模型类中定义的属性，对应该模型对照表中的一个字段；

### 1.3.2 定义属性

#### 1. 概述
- Django根据属性的类型确定以下信息：
  - 当前选择的数据库支持字段的类型；
  - 渲染管理表单时使用的默认html控件；
  - 在管理站点最低限度的验证；


- Django会为表增加自动增长的主键列;<br>每个模型只能有一个主键列；<br>如果使用选项设置某属性为主键列后，则Django不会再生成默认的主键列；


- 属性命名限制：
  - 遵循标识符规则
  - 不能是python的保留关键字；
  - 由于Django的查询方式，不允许使用连续的下划线；

#### 2. 库
- 定义属性时，需要字段类型，字段类型被定义在 django.db.models.fields 目录下；<br>为了方便使用，被导入到 django.db.models 中；


- 使用方式：
  - 导入： from django.db import models
  - 通过 models.Field 创建字段类型的对象，赋值给属性；

#### 3. 逻辑删除
- 对于重要数据都做逻辑删除，不做物理删除；<br>实现方法是定义 isDelete 属性，类型为 BooleanFiled，默认值为 False；

#### 4. 字段类型
> 字段类型(字段选项)

```
AutoField
        一个根据实际 ID 自动增长的 IntegerField，通常不指定；
        如果不指定，一个主键字段将自动添加到模型中；

CharField(max_length=字符长度)
        字符串，默认表单样式是 TextInput；

TextField
        大文本字段，一般超过4000字节使用，默认表单控件是 Textarea；

IntegerField
        整数类型；

DecimaField(max_digits=None, decimal_places=None)
        使用 python 的Decimal 实例表示的十进制浮点数；
        参数说明：
                DecimalField.max_digits
                    位数总数
                DecimalField.decimal_places
                    小数点后的数字位数

FloatField
        用 python 的 float 实例来表示的浮点数；


BooleanField
        True/False 字段，此字段的默认表单控制是 CheckboxInput；

NullBooleanField
        支持Null、True、False三种值；

DateField(auto_now=False, auto_now_add=False)
        使用 python 的 datetime.date 实例表示日期；
        参数说明：
                DateField.auto_now
                    每次保存对象时，自动设置该字段为当前时间，用于“最后一次修改”的时间戳，它总是使用当前日期，默认为 False；
                DateField.auto_now_add
                    当对象第一次被创建时自动设置当前时间，用于创建的时间戳，它总是使用当前日期，默认为 False；
        说明：
                该字段默认对应的表单控件是一个 TextInput；<br>在管理员站点添加了一个 JavaScrtpt 写的日历控件，和一个“Today”的快捷按钮；<br>包含了一个额外的 invalid_date 错误消息键；
        注意：
                auto_now、auto_now_add、default 这些设置是相互排斥的，他们之间的任何组合都将会发生错误的结果；

TimeField
        使用 python 的 datetime.time 实例表示的时间，参数同 DateField；

DateTimeField
        使用 python 的 datetime.datetime 实例表示的日期和时间，参数同 DateField；


FileField
        一个上传文件的字段；


ImageField
        继承了 FileField 的所有属性和方法，但对上传的对象进行校验，确保它是个有效的 image；
```
  
#### 5. 字段选项
> 字段类型(字段选项)

```
概述：
        通过字段选项，可以实现对字段的约束；
        在字段对象时，通过关键字参数指定；

null
        null = True     一般不写
        如果为 True，Django 将空值以 NULL 存储到数据库中，默认值是 False；

blank
        如果为 True，则该字段允许为空白，默认值是 False；

注意：null 是数据库范畴的概念；blank 是表单验证范畴的概念；


db_column
        sage = models.IntegerField(db_column='age')     指定这个字段再数据库中名字是age 
        字段的名称，如果未指定，则使用属性的名称；

db_index
        若值为 True，则在表中会为此字段创建索引；

default
        默认值

primary_key
        若为True，则该字段会成为模型的主键字段；
        
unique
        如果为True，则该字段再表中必须有唯一值；
```

#### 6. 关系
```
分类：
        Foreignkey：一对多，将字段定义在多的端中
                Grades 和 Students 是一对多关系，Foreignkey定义在Students中；
        ManyToManyField：多对多，将字段定义在两端中；
        OneToOneField：一对一，将字段定义在任意一端中；
        
        
用一访问多：
        格式： 对象.模型类小写_set
        示例： grade.student_set

用一访问一：
        格式： 对象.模型类小写
        示例： grade.student

访问id：
        格式： 对象.属性_id
        示例： student.sgrade_id
```

### 1.3.3 创建模型类

### 1.3.4 元选项
```
在模型类中定义Meta类，用于设置元信息；

        class Grade(models.Molde):
            class Meat:
                db_table='students'     # 定义数据表名，推荐使用小写字母，数据表名默认为：项目名小写_类名小写
                ordering=['id']           # 对象的默认排序字段，获取对象的列表时使用
                        ordering=['id]      # 按id字段升序排列
                        ordering=['-id']    # 按id字段降序排列
                        注意： 排序会增加数据库开销

    
示例：
        class Student(models.Model):
            # 有序字典
            GEBDER_CHOICES = (
                (0, 'male'),
                (1, 'female')    
            )
            sname = models.CharField(max_length=20)
            sgender = models.BooleanField(choices=GEBDER_CHOICES, default=0, verbose_name="性别")
            sage = models.IntegerField()
            scontend = models.TextField()
            isDelete = models.BooleanField(default=False)
            sgrade = models.ForeignKey('Grade')
        
            def __str__(self):
                return self.sname
            
            lasttime = models.DateTimeField(auto_now=True)
            createtime = models.DateTimeField(auto_now_add=True)
            
            class Meta:
                db_table = 'student'
                ordering = ['id']
```


## 1.4 模型成员
### 1.4.1 类属性
```
objects
        是Manager类型的一个对象，作用是与数据库进行交互；
        当定义模型类时，没有指定管理器，那么Django为模型创建一个名为objects的管理器；
        
        
自定义管理器：
        class Student(models.Model):
            # 自定义模型管理器
            stuObj = models.Manager()
        
        stu = Student.stuObj.get(pk=1)
        
        当为模型指定模型管理器，Django就不在为模型类生成objects模型管理器；
        
        
自定义管理器Manager类：
        模型管理器是Django的模型进行与数据库进行交互的接口；一个模型类可以有多个模型管理器；
                stuObj = models.Manager()
                stuObj1 = models.Manager()
                
        作用：
                向管理器类中添加额外的方法
                修改管理器返回的原始查询集      重写get_queryset()方法
                
        示例： 查询Student表中 isDelete=Flase的内容
                class StudentManager(models.Manager):
                    def get_queryset(self):
                        return super(StudentManager, self).get_queryset().filter(isDelete=False)
                
                class Student(models.Model):
                    # 自定义模型管理器
                    stuObj = models.Manager()
                    stuObj2 = StudentManager()
```

### 1.4.2 创建对象
```
目的：
        项数据库添加数据
        
当创建对象时，Django不会对数据库进行读写操作；
当调用save时，才与数据库有交互，将对象保存到数据库中；

注意：
        __init__() 方法已经在父类models.Model中使用，在自定义的模型中无法使用；
        
方法：
        1. 在模型类中增加一个类方法；
                class Student(models.Model):
                    # @classmethod 表明是类方法
                    @classmethod
                    def createStudents(cls, name, gender, age, contend, isD=False):
                        stu = cls(sname = name,
                                    sgender = gender,
                                    sage = age,
                                    scontend = contend,
                                    isDelete=isD)
                        return stu
            #-------------------------------------------------------------------------------
                # views.py
                def addStudent(request):
                    grade = Grade.objects.get(pk=1)
                    stu = Student.createStudents("刘德华", True, 34, '我叫刘德华', grade)
                    stu.save()
                    return HttpResponse('Yeah')
            #-------------------------------------------------------------------------------
                # urls.py
                url(r'^addstudent/$', views.addStudent),
                        
        2. 在自定义管理器中添加一个方法
                class StudentManager(models.Manager):
                    def createStudent(self, name, gender, age, contend, grade, isD=False):
                        stu = self.model()
                        stu.sname = name
                        stu.sgrade = gender
                        stu.sage = age
                        stu.scontend = contend
                        stu.sgrade = grade  
                        return stu
```

### 1.4.3 模型查询
```
概述：
        查询集表示从数据库获取的对象的集合；
        查询集可以有多个过滤器；
        过滤器就是一个函数，基于所给的参数限制查询集结果；
        从SQL角度来说，查询集和select语句等价，过滤器等同于 SQL 的 where 条件；
        

查询集：
        在管理器调用过滤器方法，返回查询集；
        查询集经过过滤器筛选后，返回新的查询集，所以可以写成链式调用；
        
        惰性执行：
                创建查询集不会带来任何数据库的访问，直到调用数据时们才会访问数据；
            
        直接访问数据的情况：
                迭代
                序列化
                与if合用
                
        返回查询集的方法称为过滤器：
                all()           返回查询集中所有的数据的对象；
                
                filter()
                        返回符合条件的数据；
                        filter(键=值, 键=值)
                        filter(键=值).filter(键=值)
                        
                exclude()       过滤掉符合条件的数据
                
                order_by()      排序
                
                values()        以条数据就是一个字典，返回一个列表；
                
        返回单个数据：
                get()
                        返回一个满足条件的对象；
                        注意：
                                如果没有找到符合条件的对象，会引发 "模型类.DoesNotExist" 异常；
                                如果找到多个对象，会引发 "模型类.MultipleObjectsReturned" 异常；
                count()     返回当前查询集中的个数；
                first()     返回当前查询集中的第一个对象；
                last()      返回当前查询集中的最后一个对象；
                exists()    判断查询集中是否有数据，有则返回True;
                
        限制查询集：
                查询集返回列表，可以使用下标来进行限制，等同于 SQL 的 limit 语句；
                        def student3(request):
                            studentList = Student.stuObj.all()[0:3]
                            return render(request, 'myapp/student.html', {'student': studentList})
                
                注意：下标不能是负数；
                
                需求：分页显示学生信息
                url.py
                    url(r'^stupage/(\d)/$', views.stupage),
                    
                views.py
                    def stupage(request, page):
                        # 0-3  3-6  6-9
                        #  1    2    3
                        page = int(page)
                        studentList = Student.stuObj.all()[(page-1)*3: (page*3)]
                        return render(request, 'myapp/student.html', {'student': studentList})
        
        查询集的缓存：
                概述：
                        每个查询集都包含一个缓存，来最小化对数据库访问；
                        在新建的查询集中，缓存首次为空，第一次对查询集求值，会发生数据缓存；
                        Django会将查询的数据做一个缓存，并返回查询结果，以后查询直接使用查询集的缓存；
        
        字段查询：
                概述：
                        实现了 SQL 中的 where 语句，作为方法filter()、exclude()、get()的参数
                        语法：  
                                属性名称__比较运算符=值
                        外键：  
                                属性名_id
                        转义：  
                                like 语句使用%是为了匹配占位，匹配数据中的% (where like '\%')
                                filter(sanme_cotains='%')
                                
                比较运算符：
                        exact
                                判断，大小写敏感（区分大小写）
                                filter(isDelete=False)
                        
                        contans
                                是否包含，大小写敏感
                                studentList = Student.stuObj.all().filter(sname__contains='张')
                        
                        startswith、endswith
                                以value开头或者结尾，大小写敏感
                                studentList = Student.stuObj.all().filter(sname__startswith='张')
                        
                    以上四个在参数前加 i 就表示不区分大小写：iexact、icontans、istartswith、iendswith
                    
                        isnull、isnotnull
                                是否为空
                            
                        in
                                是否包含在范围内
                                studentList = Student.stuObj.all().filter(pk__in=[2,4,6,8])
                        
                        gt      大于
                        gte     大于等于
                        lt      小于
                        lte     小于等于
                                studentList = Student.stuObj.all().filter(sage__gt=20)
                                
                        year
                        month
                        day
                        week_day
                        hour
                        minute
                        second
                                给datetime、date、time类型的字段使用
                                
                        跨关联查询
                                处理join查询
                                        语法：
                                                模型类名__属性名__比较运算符
                                                grade = Grade.objects.filter(student__scontend__contains='刘德华')
                                                print(grade)
                                                描述中带有刘德华的班级的内容
                        
                        查询快捷
                                pk      代表主键
                    
                        
                聚合函数：
                        使用aggregate()函数返回聚合函数的值；
                        Avg
                        Count
                        Max
                        Min
                        Sum
                                from django.db.models import Max
                                def studentSearch(request):
                                    maxAge = Student.stuObj.aggregate(Max('sage'))
            
                F对象
                        可以使用模型的 A属性 与 B属性 进行比较
                                from django.db.models import F
                                def grades(request):
                                    g = Grades.object.filter(ggirlnum__gt=F('gboynum'))
                                    print(g)
                                return HttpResponse('女生比男生数多的班')
                        支持F对象的算数运算
                                g = Grades.object.filter(ggirlnum__gt=F('gboynum')+10)
                        
                        支持F对象的时间的运算
                
                Q对象
                        概述：  过滤器的方法中的关键字参数，条件为and模式
                        需求：  进行or查询
                        解决：  使用Q对象
                                studentList = Student.stuObj2.filter(Q(pk__lt=5) | Q(sage__gt=20))
                                studentList = Student.stuObj2.filter(Q(pk__lt=5))       只有一个Q对象，就是用于匹配的；
                                studentList = Student.stuObj2.filter(~Q(pk__lt=5))      ~   取反
```

### 1.4.4 删、改
```
删
        对象名.del()
改
        对象名.save()
```




