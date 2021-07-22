[toc]

# 1. 环境部署

> 安装drf框架

```shell
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple django pymysql

pip install -i https://pypi.tuna.tsinghua.edu.cn/simple djangorestframework
```

> 添加rest_framework应用

```python
INSTALLED_APPS = [
	...
    'rest_framework',
]
```



# 2. Serializer序列化器

> 模型类 books01/models.py 

```python
class BookInfo(models.Model):
    name = models.CharField(max_length=20, verbose_name='书名')
    pub_date = models.DateField(verbose_name='发布日期')
    read_count = models.IntegerField(default=0, verbose_name="阅读量")
    comment_count = models.IntegerField(default=0, verbose_name="评论量")
    is_delete = models.BooleanField(default=False, verbose_name='逻辑删除')
    
    class Meta:
        db_table = 'tb_bookinfo'
        verbose_name = '图书'
```

> 序列化器 books01/serializer.py

```python
from rest_framework import serializers

class BookInfoSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=20, label='书名')
    pub_date = serializers.DateField(label='发布日期')
    read_count = serializers.IntegerField(default=0, label='阅读量')
    comment_count = serializers.IntegerField(default=0, label='评论量')
    is_delete = serializers.BooleanField(default=False, label='逻辑删除')
```

> 添加数据

```sql
insert into books01.tb_bookinfo (name,pub_date,read_count,comment_count,is_delete) 
values 
('西游记','1876-04-05', 107, 98, False),
('水浒传', '1762-06-09', 96, 20, False),
('骆驼祥子', '1962-09-08', 60, 25, False),
('天龙八部', '1983-07-01', 201, 9, False);
```



## 2.1 查询

> books01/urls.py

```python
urlpatterns = [
    path(r'books01/', BooksInfoView.as_view()),
    re_path(r'book01/(?P<id>\d+)/', BookInfoView.as_view()),
]
```

### 2.1.1 多值查询

> books01/views.py

```
class BooksInfoView(View):
    # GET http://127.0.0.1:8000/books01/
    def get(self, request):
        books = BooksInfo.objects.all()
        serializer = BooksInfoSerializer(instance=books, many=True)
        return JsonResponse(status=200, data=serializer.data, safe=False)
```

在创建序列化器时使用`many`参数, 以列表形式输出多行数据;

### 2.1.2 单值查询

> books01/views.py

```
class BookInfoView(View):
    # GET http://127.0.0.1:8000/book01/1/
    def get(self, request, id):
        book = BooksInfo.objects.get(pk=id)
        serializer = BooksInfoSerializer(instance=book)
        return JsonResponse(status=200, data=serializer.data)
```

### 2.1.3 关联查询

> 补充模型类 books01/models.py 

```
class HeroInfo(models.Model):
    GENDER_CHOICES = (
        (0, 'male'),
        (1, 'female')
    )
    name = models.CharField(max_length=20, verbose_name='姓名')
    gender = models.SmallIntegerField(choices=GENDER_CHOICES, default=0, verbose_name='性别')
    book = models.ForeignKey(BookInfo, on_delete=models.CASCADE, verbose_name='图书')
    is_delete = models.BooleanField(default=False, verbose_name='逻辑删除')
```

> 添加数据

```sql
insert into books01.tb_heroinfo (name,gender,book_id,is_delete)
values
('孙悟空', 0, 1, False),
('白骨精', 1, 1, False),
('李逵', 0, 2, False);
```



> 补充序列化器 books01/serializer.py

```

```



## 创建

































