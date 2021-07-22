[toc]

# 1 工程搭建

## 1.1 环境配置

```shell
[root@localhost ~]# mkdir Flask01
[root@localhost ~]# cd Flask01/

[root@localhost Flask01]# python3 -m venv venv

[root@localhost Flask01]# . venv/bin/activate
(venv) [root@localhost Flask01]# pip install flask

(venv) [root@localhost Flask01]# pip list
Package      Version
------------ -------
click        7.1.2
Flask        1.1.2
itsdangerous 1.1.0
Jinja2       2.11.3
MarkupSafe   1.1.1
pip          20.1.1
setuptools   47.1.0
Werkzeug     1.0.1
```

> 常用扩展

http://flask.pocoo.org/extensions

| Flask-SQLalchemy     | 数据库ORM                     |
| -------------------- | ----------------------------- |
| Flask-script         | 插入脚本                      |
| Flask-migrate        | 管理迁移数据库                |
| Flask-Session        | Session存储方式指定           |
| Flask-WTF            | 表单                          |
| Flask-Mail           | 邮件                          |
| Flask-Bable          | 提供国际化和本地化支持, 翻译  |
| Flask-Login          | 认证用户状态                  |
| Flask-OpenID         | 认证                          |
| <b>Flask-RESTful</b> | <b>开发REST API的工具</b>     |
| Falsk-Bootstrap      | 集成前端Twitter Bootstrap框架 |
| Flask-Moment         | 本地化日期和时间              |
| Flask-Admin          | 简单而且可扩展的接口管理框架  |



## 1.2 HelloWorld

> app.py

```python
from flask import Flask

app = Flask(__name__)


@app.route('/')
def index():
    return 'hello flask'


if __name__ == '__main__':
    app.run()
```



## 1.3 参数说明

### 1.3.1 项目初始化参数

- import_name
  - Flask程序所在的包(模块)，传 `__name__` 就可以
    - HelloWord程序中`__name__` 就是 `__main__`, 会把`__main__`程序所在的目录当做程序主目录
  - 其可以决定 Flask 在访问静态文件时查找的路径
- static_url_path
  - 静态文件访问路径，可以不传，默认为：`/ + static_folder`
- static_folder
  - 静态文件存储的文件夹，可以不传，默认为 `static`
- template_folder
  - 模板文件存储的文件夹，可以不传，默认为 `templates`

#### 修改参数

```
app = Flask(__name__, static_url_path='/s', static_folder='staticdir')
```

> 目录结构

```
Flask01
├── app.py
└── staticdir
    └── a.png
```

> 访问URL

```
http://127.0.0.1:5000/s/a.png
```

### 1.3.2 应用程序参数

Flask将配置信息保存到了app.config属性中，该属性可以按照字典类型进行操作

#### 设置参数

> 从配置对象中加载 app.config.from_object(配置类名)

```
# 定义配置类
class DefaultConfig(object):
    DBNAME = 'book01'
# 加载配置
app.config.from_object(DefaultConfig)
```

> 从配置文件加载 app.config.from_pyfile('配置文件路径')

```
Flask01
├── app.py
├── settings.py
└── staticdir
    └── a.png

settings.py
	DBHOST = '127.0.0.1'
```

```
app.config.from_pyfile('settings.py')
# app.config.from_pyfile('conf/settings.py')
```

> 从环境变量加载 app.config.from_envvar('环境变量名')

Flask使用环境变量加载配置的本质是**通过环境变量值找到配置文件**，再读取配置文件的信息

```
先设置环境变量
	windows: SET CONF_FILE=settings.py

app.config.from_envvar('CONF_FILE')
```

**slient** 表示系统环境变量中没有设置相应值时是否抛出异常

- True  表示环境变量不存在时, Flask不报错
- False 表示环境变量不存在时, Flask抛出`RuntimeError`异常

#### 读取参数

```
print(app.config.get('DBNAME'))
print(app.config['DBHOST'])
```

#### 应用场景

|                  | 优点                                           | 缺点                     | 场景     |
| ---------------- | ---------------------------------------------- | ------------------------ | -------- |
| 从配置对象中加载 | 配置继承, 配置服用                             | 敏感信息暴露在代码中     | 默认配置 |
| 从配置文件加载   | 独立文件, 保护敏感数据                         | 配置文件目录在代码中写死 |          |
| 从环境变量加载   | 独立文件, 保护敏感数据<br>文件路径不固定, 灵活 | 不方便, 需要设置环境变量 |          |



### 1.3.3 app.run参数

指定运行时监听的 IP, 端口, 是否开启调试模式

```
app.run(host='0.0.0.0', port=8000, debug=True)
```



## 1.4 flask新版运行方式

- 代码中不需要写app.run, 通过`flask run`运行
- 通过`FLASK_APP`环境变量指定运行的app(FLASK_APP=app.py)

- flask run -h 0.0.0.0 -p 8000 绑定ip和端口
- FLASK_ENV 指定flask运行模式
  - FLASK_ENV=production 运行在生产模式, 默认
  - FLASK_ENV=devlopment 运行在开发模式

> python -m flask run



# 2 路由和蓝图

## 2.1 路由

### 2.1.1 查看路由信息

> 命令行查看

```
(venv) D:\alec.taikang\03_Projects\PythonWeb\Flask01>flask routes
Endpoint  Methods  Rule
--------  -------  ------------------
index     GET      /
static    GET      /s/<path:filename>
```

> 代码中查看

```
print(app.url_map)

Map([<Rule '/' (GET, HEAD, OPTIONS) -> index>, <Rule '/s/<filename>' (GET, HEAD, OPTIONS) -> static>])
```

> 构造web接口显示路由信息

```
@app.route('/')
def url_map():
    rules_iterator = app.url_map.iter_rules()
    return json.dumps({rule.endpoint: rule.rule for rule in rules_iterator})
```

### 2.1.2 指定请求方式

Flask中, 路由的默认请求方式:

- GET
- OPTIONS(自带): 简化版的GET, 用于询问服务器接口信息
- HEAD(自带): 简化版的GET, 只返回GET请求处理时的响应头, 不返回响应体

使用`methods`指定请求方式

```
@app.route('/', methods=['POST', 'PUT', 'DELETE', 'PATCH'])
def hello_world():
    return 'Hello World!'
```

```
(venv) D:\Projects\Python\alec\Flask01>flask routes
Endpoint     Methods                   Rule
-----------  ------------------------  -----------------------
hello_world  DELETE, PATCH, POST, PUT  /
static       GET                       /static/<path:filename>
```



## 2.2 蓝图

### 2.2.1 创建蓝图

#### 单文件创建蓝图

```
from flask import Flask, Blueprint

app = Flask(__name__)

# 创建蓝图对象
user_bp = Blueprint('user', __name__)

# 在蓝图对象上进行操作,注册路由,指定静态文件夹,注册模版过滤器
@user_bp.route('/profile')
def get_profile():
    return 'user profile'

# 注册蓝图
# app.register_blueprint(user_bp)  # http://127.0.0.1:5000/profile
# 增加/user路由前缀
app.register_blueprint(user_bp, url_prefix='/user')  # http://127.0.0.1:5000/user/profile
```

#### 多文件创建蓝图

> 目录结构

```
Flask01
├── app.py
└── goods
    ├── __init__.py
    └── views.py
```

> goods/\_\_init\_\_.py

```
from flask import Blueprint

goods_bp = Blueprint('goods', __name__)

from . import views
```

> goods/views.py

```
from . import goods_bp

@goods_bp.route('/goods')
def get_goods():
    return 'get goods'
```

> app.py

```
from flask import Flask, Blueprint
from goods import goods_bp
app = Flask(__name__)

app.register_blueprint(goods_bp)  # http://127.0.0.1:5000/goods
```

```
(venv) D:\Projects\Python\alec\Flask01>flask routes
Endpoint          Methods  Rule
----------------  -------  -----------------------
goods.get_goods   GET      /goods
static            GET      /static/<path:filename>
user.get_profile  GET      /user/profile
```

### 2.2.2 蓝图使用静态文件

需要在初始化的时候设置, 同项目初始化

> 项目初始化

```
app = Flask(__name__, static_url_path='/s', static_folder='staticdir')
```

> 蓝图初始化 goods/\_\_init\_\_.py

```
goods_bp = Blueprint('goods', __name__, static_folder='static')
```



# 3 请求和响应

## 3.1 处理请求

```
请求报文
GET /path?a=1 HTTP/1.1
Content-Type: application/json
...
body -> file  form  json  xml
```

### 3.1.1 URL路径参数

```
# @app.route('/users/<转换器:参数>')
@app.route('/users/<user_id>') # user_id默认是string类型
def user_info(user_id):
    return user_id
```

Flask提供的转换器类型:

```
DEFAULT_CONVERTERS = {
    'default':          UnicodeConverter,
    'string':           UnicodeConverter,
    'any':              AnyConverter,
    'path':             PathConverter,
    'int':              IntegerConverter,
    'float':            FloatConverter,
    'uuid':             UUIDConverter,
}
```

#### 自定义转换器

```
from flask import Flask
from werkzeug.routing import BaseConverter

# 1. 创建转换器类(regex是BaseConverter的属性)
class MobileConverter(BaseConverter):
    regex = '1[3-9]\d{9}'

# 2. 将自定义转换器添加到转换器字典中，并指定转换器使用时名字为: mobile
app = Flask(__name__)
app.url_map.converters['mobile'] = MobileConverter

# 3. 使用自定义转换器
@app.route('/mobile/<mobile:mob_num>')
def send_sms_code(mob_num):
    return 'send sms code to %s' % mob_num
```



### 3.1.2 其他参数(request对象)

不同位置的参数都存放在request的不同属性中

| 属性    | 说明                           | 类型           |
| :------ | :----------------------------- | :------------- |
| data    | 记录请求的数据，并转换为字符串 | *              |
| form    | 记录请求中的表单数据           | MultiDict      |
| args    | 记录请求中的查询参数           | MultiDict      |
| cookies | 记录请求中的cookie信息         | Dict           |
| headers | 记录请求中的报文头             | EnvironHeaders |
| method  | 记录请求使用的HTTP方法         | GET/POST       |
| url     | 记录请求的URL地址              | string         |
| files   | 记录请求上传的文件             | *              |

#### 获取GET请求参数

获取GET请求`/users?user_id=1`中`user_id`的值

```
from flask import Flask
from flask import request

app = Flask(__name__)


@app.route('/users')
def user_info():
    id = request.args.get('user_id')
    return 'user id = %s' % id
```

#### JSON数据

```
@app.route('/users', methods=['POST'])
def register():
    user_info = request.data
    print(user_info)
    return 'ok'
```

#### 上传文件

> 前端代码(表单)

```
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="pic">
    <input type="submit" value="上传">
</form>
```

> 后端代码

```
from flask import Flask
from flask import request
from flask import render_template

app = Flask(__name__)

@app.route('/')
def index():
    """上传页面"""
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload_file():
    f = request.files['pic']
    f.save('./demo.png')
    return 'upload ok'
```



## 3.2 响应处理

```
响应报文
HTTP/1.1 200 OK
Content-Type: application/json # 响应头描述body格式
...
body
```





### 3.2.1 返回模板 - render_template

使用render_template返回模板文件, 参考`文件上传`index方法

### 3.2.2 重定向 - redirect

```
from flask import redirect

@app.route('/baidu')
def jump():
    return redirect('https://www.baidu.com')
```

### 3.2.3 返回JSON - jsonify

```
from flask import jsonify

@app.route('/demo3')
def demo3():
    json_dict = {'name': 'Jack', 'age': 22}
    return jsonify(json_dict)
```

### 3.2.4 返回状态码和响应头

#### 通过元组返回(return多值)

元组格式必须是 **(response, status, headers)**

```
@app.route('/demo4')
def demo4():
    return 'demo4', 666, {'name': 'alec'}
```

#### 通过make_response返回

````
from flask import make_response

@app.route('/demo5')
def demo5():
    resp = make_response('demo5')
    resp.status = '888' # 必须是字符串, 见base_response.py > @status.setter
    resp.headers['class'] = 'python'
    return resp
````



## 3.3 Cookie和Session

### 3.3.1 Cookie

#### 设置

```
@app.route('/demo6')
def demo6():
    resp = make_response('set cookie ok')
    # resp.set_cookie('username', 'alec') # 设置cookie
    # resp.headers['Set-Cookie'] = 'password=66666;Max-Age=30' # cookie的本质是Response Headers中的键值对
    resp.set_cookie('password', '123123', max_age=30)  # 设置cookie+过期时间(max_age 单位:s)
    return resp
```

#### 读取

```
@app.route('/demo7')
def demo7():
    value = request.cookies.get('password')
    return 'cookie password value = {}'.format(value)
```

### 3.3.2 Session

使用session需要先设置secret_key

```
app.secret_key = 'alec'

或者

class DefaultConfig(object):
    SECRET_KEY = 'alec'
   
app.config.from_object(DefaultConfig)
```

#### 设置

```
from flask import session

@app.route('/demo8')
def demo8():
    session['username'] = 'bob'
    return 'set session ok'
```

#### 读取

```
@app.route('/demo9')
def demo9():
    username = session.get('username')
    return 'get session username = {}'.format(username)
```

> 说明

Flask默认session是存储在浏览器Cookie的session字段里, Flask使用SECRET_KEY做签名



# 4 请求钩子与上下文

## 4.1 异常处理

### 4.1.1 HTTP主动抛出异常 - abort

使用abort主动抛出异常

```
from flask import abort

@app.route('/demo10')
def demo10():
    abort(404)
```

### 4.1.2 捕获错误 - errorhandler

- 使用`errorhandler`装饰器, 捕获错误
  
- 注册一个错误处理程序，当程序抛出指定错误状态码的时候，就会调用该装饰器所装饰的方法
  
- 参数

    - HTTP的错误状态码

    ```
    访问 /demo10 页面显示 '资源不存在'
    @app.errorhandler(404)
    def demo11(error):	# error是异常对象
        return '资源不存在'
    ```

    - 指定异常

    ```
    # 抛出异常
    @app.route('/demo12')
    def demo12():
        raise ConnectionError('连接数据库异常')
    
    # 捕获异常
    @app.errorhandler(ConnectionError)
    def demo13(error):
        return '连接数据库异常, 开始重新连接'
    ```



## 4.2 请求钩子

在客户端和服务器交互的过程中，有些准备工作或扫尾工作需要处理，比如：

- 在请求开始时，建立数据库连接；
- 在请求开始时，根据需求进行权限校验；
- 在请求结束时，指定数据的交互格式；

为了让每个视图函数避免编写重复功能的代码，Flask提供了通用设施的功能，即请求钩子。

请求钩子是通过装饰器的形式实现，Flask支持如下四种请求钩子：

| before_first_request | 在处理第一个请求前执行                                       |
| -------------------- | ------------------------------------------------------------ |
| before_request       | 在每次请求前执行<br>如果在某修饰的函数中返回了一个响应，视图函数将不再被调用 |
| after_request        | 如果没有抛出错误，在每次请求后执行<br>接受一个参数：视图函数作出的响应<br>在此函数中可以对响应值在返回之前做最后一步修改处理<br>需要将参数中的响应在此参数中进行返回 |
| teardown_request     | 在每次请求后执行<br>接受一个参数：错误信息，如果有相关错误抛出 |

```
from flask import Flask
from flask import abort

app = Flask(__name__)

# 在第一次请求之前调用，可以在此方法内部做一些初始化操作
@app.before_first_request
def before_first_request():
    print('1 - before_first_request')

# 在每一次请求之前调用，这时候已经有请求了，可以在这个方法里面做请求的校验
@app.before_request
def before_request():
    print('2 - before_request')

# 在执行完视图函数之后会调用，并且会把视图函数所生成的响应传入,可以在此方法中对响应做最后一步统一的处理
@app.after_request
def after_request(response):
    print('3 - after_request: {}'.format(response))
    return response

# 请每一次请求之后都会调用，会接受一个参数，参数是服务器出现的错误信息
@app.teardown_request
def teardown_request(error):
    print('4 - teardown_request: {}'.format(error))

@app.route('/')
def index():
    return 'index'

@app.route('/pages/1')
def page1():
    return 'page1'

@app.route('/pages/2')
def page2():
    abort(404)
```

```
GET http://127.0.0.1:5000
1 - before_first_request
2 - before_request
3 - after_request: <Response 5 bytes [200 OK]>
4 - teardown_request: None

GET http://127.0.0.1:5000
2 - before_request
3 - after_request: <Response 5 bytes [200 OK]>
4 - teardown_request: None

GET http://127.0.0.1:5000/pages/1
2 - before_request
3 - after_request: <Response 5 bytes [200 OK]>
4 - teardown_request: None

GET http://127.0.0.1:5000/pages/2
2 - before_request
3 - after_request: <Response streamed [404 NOT FOUND]>
4 - teardown_request: None
```

## 4.3 上下文

Flask中有两种上下文，请求上下文和应用上下文

Flask中上下文对象：相当于一个容器，保存了 Flask 程序运行过程中的一些信息。

### 4.3.1 请求上下文(request context)

在 flask 中，可以直接在视图函数中使用 **request** 这个对象进行获取相关数据，而 **request** 就是请求上下文的对象，保存了当前本次请求的相关数据，请求上下文对象有：`request`、`session`

- request
    - 封装了HTTP请求的内容，针对的是http请求。举例：user = request.args.get('user')，获取的是get请求的参数。
- session
    - 用来记录请求会话中的信息，针对的是用户信息。举例：session['name'] = user.id，可以记录用户信息。还可以通过session.get('name')获取用户信息

### 4.3.2 应用上下文(application context)

它的字面意思是 应用上下文，但它不是一直存在的，它只是request context 中的一个对 app 的代理(人)，所谓local proxy。它的作用主要是帮助 request 获取当前的应用，它是伴 request 而生，随 request 而灭的。

应用上下文对象有：`current_app`，`g`

#### current_app

应用程序上下文,用于存储应用程序中的变量，可以通过current_app.name打印当前app的名称，也可以在current_app中存储一些变量，例如：

- 应用的启动脚本是哪个文件，启动时指定了哪些参数
- 加载了哪些配置文件，导入了哪些配置
- 连了哪个数据库
- 有哪些public的工具类、常量
- 应用跑再哪个机器上，IP多少，内存多大

> `current_app` 就是当前运行的flask app，在代码不方便直接操作flask的app对象时，可以操作`current_app`就等价于操作flask app对象

```
app1 = Flask(__name__)
app2 = Flask(__name__)

app1.redis_cli = 'app1 redis client'
app2.redis_cli = 'app2 redis client'


@app1.route('/app1/1')
def app1_1():
    return current_app.redis_cli


@app1.route('/app1/2')
def app1_2():
    return current_app.redis_cli


@app2.route('/app2/1')
def app2_1():
    return current_app.redis_cli


@app2.route('/app2/2')
def app2_2():
    return current_app.redis_cli
```

运行app1

```
GET http://127.0.0.1:5000/app1/1   --->   app1 redis client
GET http://127.0.0.1:5000/app1/2   --->   app1 redis client
```

运行app2

```
GET http://127.0.0.1:5000/app2/1   --->   app2 redis client
GET http://127.0.0.1:5000/app2/2   --->   app2 redis client
```

#### g

g 作为 flask 程序全局的一个临时变量，充当中间媒介的作用，我们可以通过它在一次请求调用的多个函数间传递一些数据。每次请求都会重设这个变量。

```
from flask import Flask, g

app = Flask(__name__)

def db_query():
    user_id = g.user_id
    user_name = g.user_name
    print('id={}, name={}'.format(user_id, user_name))

@app.route('/users/<int:user_id>')
def user_info(user_id):
    g.user_id = user_id
    g.user_name = 'alec'
    db_query()
    return 'g object'
```

#### app_context 和 request_context

在Flask没有运行的时候(ipython环境中), 调用`current_app`, `g`会报错

```
>>> from flask import Flask
>>> app = Flask('')
>>> app.redis_cli = 'redis client'
>>> from flask import current_app
...
    raise RuntimeError(_app_ctx_err_msg)
RuntimeError: Working outside of application context.
```

> app_context

`app_context`为我们提供了应用上下文环境，允许我们在外部使用应用上下文`current_app`、`g`

```
# 借助with语句使用app_context创建应用上下文
>>> with app.app_context():
...     print(current_app.redis_cli)
...     
redis client
```

> request_context

`request_context`为我们提供了请求上下文环境，允许我们在外部使用请求上下文`request`、`session`

```
>>> from flask import Flask
>>> app = Flask('')
>>> request.args  # 错误，没有上下文环境
...报错
>>> environ = {'wsgi.version':(1,0), 'wsgi.input': '', 'REQUEST_METHOD': 'GET', 'PATH_INFO': '/', 'SERVER_NAME': 'itcast server', 'wsgi.url_scheme': 'http', 'SERVER_PORT': '80'}  # 模拟解析客户端请求之后的wsgi字典数据
>>> with app.request_context(environ):  # 借助with语句使用request_context创建请求上下文
...     print(request.path)
... 
```



# 5 Flask-RESTful

## 5.1  工程搭建

### 5.1.1 安装

```
pip install flask-restful
```

### 5.1.2 Hello World

```
from flask import Flask
from flask_restful import Resource, Api

app = Flask(__name__)
api = Api(app)

# 定义视图类
class HelloWorldResource(Resource):
    def get(self):
        return {'msg': 'hello world'}

    def post(self):
        return {'msg': 'ok'}

api.add_resource(HelloWorldResource, '/')
```

## 5.2 视图

### 5.2.1 路由别名

用途不详

```
api.add_resource(HelloWorldResource, '/', endpoint='hello')
```

### 5.2.2 蓝图

```
from flask import Flask, Blueprint
from flask_restful import Resource, Api

app = Flask(__name__)
api = Api(app)

user_bp = Blueprint('user', __name__)
user_api = Api(user_bp)

class UserProfileesource(Resource):
    def get(self):
        return {'msg': 'hello alec'}

    def post(self):
        return {'msg': 'ok'}

user_api.add_resource(UserProfileesource, '/users')
app.register_blueprint(user_bp)
```



### 5.2.3 视图类装饰器

使用`method_decorators`属性, 给视图类添加装饰器

> 为类视图中的所有方法添加装饰器

```
def decorator1(func):
    def wrapper(*args, **kwargs):
        print('decorator1')
        return func(*args, **kwargs)
    return wrapper


def decorator2(func):
    def wrapper(*args, **kwargs):
        print('decorator2')
        return func(*args, **kwargs)
    return wrapper

class DemoResource(Resource):
    method_decorators = [decorator1, decorator2]	# decorator2 优先于 decorator1 执行
	
	#等价于:
	#@decorator2
	#@decorator1
    def get(self):
        return {'msg': 'get view'}

    def post(self):
        return {'msg': 'post view'}
        
api.add_resource(DemoResource, '/')
```

> 为类视图中不同的方法添加不同的装饰器

```
class DemoResource(Resource):
    method_decorators = {
        'get': [decorator1, decorator2],
        'post': [decorator1]
    }

    # 使用了decorator1 decorator2两个装饰器
    def get(self):
        return {'msg': 'get view'}

    # 使用了decorator1 装饰器
    def post(self):
        return {'msg': 'post view'}

    # 未使用装饰器
    def put(self):
        return {'msg': 'put view'}
```



## 5.3 请求

Flask-RESTful 提供了`RequestParser`类，用来检验和转换请求数据;

```
from flask_restful.reqparse import RequestParser

app = Flask(__name__)
api = Api(app)

class DemoResource(Resource):
    def get(self, pk):
        # name = request.args.get('name')
        # age = request.args.get('age')

        # 1. 创建RequestParser对象
        parse = RequestParser()

        # 2. 向RequestParser中添加需要检验或转换的参数
        parse.add_argument('name')
        parse.add_argument('age', type=int)

        # 3. 使用parse_args方法启动检验处理
        args = parse.parse_args()

        # 4. 检验之后从检验结果中获取参数时, 可按照字典操作或对象属性操作
        print(args)
        return {'pk': pk, 'name': args.name, 'age': args['age']}

api.add_resource(DemoResource, '/<int:pk>')
```

```
GET http://127.0.0.1:5000/1/?name=alec&age=10

{
    "pk": 1,
    "name": "alec",
    "age": 10
}
```

### add_argument参数说明

> required

描述请求是否一定要携带对应参数，**默认值为False**

- True 强制要求携带:  若未携带，则校验失败，向客户端返回错误信息，状态码400
- False 不强制要求携带:  若不强制携带，在客户端请求未携带参数时，取出值为None

> help

参数检验错误时返回的错误描述信息

> action

描述对于请求参数中出现多个同名参数时的处理方式

- `action='store'` 保留出现的第一个， 默认
- `action='append'` 以列表追加保存所有同名参数的值

> type

描述参数应该匹配的类型，可以使用python的标准数据类型string、int，也可使用Flask-RESTful提供的检验方法，还可以自己定义

- 标准类型: `str`, `int`, `float`, `bool`

- Flask-RESTful提供

  检验类型方法在`flask_restful.inputs`模块中

  - `url`

  - `regex`(指定正则表达式)

    ```
    from flask_restful import inputs
    rp.add_argument('a', type=inputs.regex(r'^\d{2}&'))
    ```

  - `natural` 自然数0、1、2、3...

  - `positive` 正整数 1、2、3...

  - `int_range(low ,high)` 整数范围

  - `boolean`

- 自定义

  ```
  def mobile(mobile_str):
      """
      检验手机号格式
      :param mobile_str: str 被检验字符串
      :return: mobile_str
      """
      if re.match(r'^1[3-9]\d{9}$', mobile_str):
          return mobile_str
      else:
          raise ValueError('{} is not a valid mobile'.format(mobile_str))
  
  rp.add_argument('a', type=mobile)
  ```

> location

描述参数应该在请求数据中出现的位置: `form`, `args`, `headers`, `cookies`, `json`, `files`

```
# Look only in the POST body
parser.add_argument('name', type=int, location='form')

# Look only in the querystring
parser.add_argument('PageSize', type=int, location='args')

# From the request headers
parser.add_argument('User-Agent', location='headers')

# From http cookies
parser.add_argument('session_id', location='cookies')

# From json
parser.add_argument('user_id', location='json')

# From file uploads
parser.add_argument('picture', location='files')
```

也可指明多个位置

```
parser.add_argument('text', location=['headers', 'json'])
```



## 5.4 响应

### 5.4.1 序列化

Flask-RESTful 提供了marshal工具，用来帮助我们将数据序列化为特定格式的字典数据，以便作为视图的返回值

- marshal_with装饰器
- marshal方法

```
from flask import Flask
from flask_restful import Resource, Api
from flask_restful import fields, marshal_with, marshal

"""
    fields : 用于声明数据类型
"""
app = Flask(__name__)
api = Api(app)

user_info = {
    'pk': fields.Integer,
    'name': fields.String,
    'age': fields.Integer
}

# 模拟数据库中查到的数据
class User(object):
    def __init__(self, pk, name, age):
        self.pk = pk
        self.name = name
        self.age = age

# 使用marshal_with装饰器序列化
class DemoResource1(Resource):
    @marshal_with(user_info)
    def get(self, pk):
        user = User(pk, 'alec', 18)
        return user

# 使用marshal方法序列化
class DemoResource2(Resource):
    def get(self, pk):
        user = User(pk, 'alec', 22)
        return marshal(user, user_info)

api.add_resource(DemoResource1, '/demo1/<int:pk>')
api.add_resource(DemoResource2, '/demo2/<int:pk>')
```

### 5.4.2 定制返回的JSON格式

> 需求

想要接口返回的JSON数据具有如下统一的格式

```
{"message": "描述信息", "data": {要返回的具体数据}}
```

在接口处理正常的情况下， message返回ok即可

但是想在每个接口正确返回时省略message字段

```
class DemoResource(Resource):
    def get(self):
        return {'user_id':1, 'name': 'itcast'}
```

对于诸如此类的接口，能否在某处统一格式化成上述需求格式

```
{"message": "OK", "data": {'user_id':1, 'name': 'itcast'}}
```

> 解决方案

Flask-RESTful的Api对象提供了一个`representation`的装饰器，允许定制返回数据的呈现格式

```
@api.representation('application/json')  # 参数为响应数据的格式
def handler(data, code, headers):
    """
    :param data: 响应体中的数据
    :param code: 状态码
    :param headers: 响应头
    """
    # 自定义预处理
    # 拼接响应
    resp = make_response()
    resp.headers.extend(headers)
    # 返回响应
    return resp
```



```
from flask import Flask, make_response
from flask_restful import Resource, Api
from flask_restful import fields, marshal_with, marshal
from json import dumps

app = Flask(__name__)
api = Api(app)

user_info = {
    'pk': fields.Integer,
    'name': fields.String,
    'age': fields.Integer
}

# 模拟数据库中查到的数据
class User(object):
    def __init__(self, pk, name, age):
        self.pk = pk
        self.name = name
        self.age = age

class DemoResource(Resource):
    @marshal_with(user_info)
    def get(self, pk):
        user = User(pk, 'alec', 18)
        return user

# 代码参考(flask_restful.representations.json)
@api.representation('application/json')  # 参数为响应数据的格式
def handler(data, code, headers=None):
    if 'message' not in data:
        data = {
            'message': 'ok',
            'data': data
        }
    dumped = dumps(data)
    resp = make_response(dumped, code)
    resp.headers.extend(headers)
    # 返回响应
    return resp

api.add_resource(DemoResource, '/demo/<int:pk>')
```

```
GET http://127.0.0.1:5000/demo/1

{"message": "ok", "data": {"pk": 1, "name": "alec", "age": 18}}
```















































