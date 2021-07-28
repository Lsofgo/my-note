# 6 API的使用

## 6.1 API概述
计算机一次Requests请求和服务器端的Response回应， 即实现了互联网， 而API也是通过Requests请求和服务器端的Response回应来完成API的一次调用。<br>
API的请求使用非常严谨的语法， 并且API返回的是JSON或XML格式的数据， 而不是HTML数据。<br>

## 6.2 API使用方法
API是通过Requests请求和服务器端的Response回应来完成API的一次调用；<br>
所以用Python语言进行API的调用时，便可以使用Requests库来进行请求；
除了使用Requests库的GET方法， 还需了解POST、 PUT和DELETE方法。
- POST方法就是填写表单或提交信息时所做的事情， 如登录一个网址， 使用的便是POST方法。
- PUT方法在API里有时会用到。 PUT请求是用来更新一个对象或信息。 对于老用户的个人信息进行更新时就会用到PUT方法。
- DELETE方法用于删除一个对象， 但在公共API中并不常见， 毕竟一个公司不会让其他人随便地删除数据库中的信息。

## 6.3 API验证
后面说

## 6.4 解析JSON数据
调用API， 服务器会返回JSON格式的数据，需要从JSON中提取有用信息；
这样解析Jason格式：
```
import json

jsonstring = '{"user_man":[{"name":"Peter"},{"name":"xiaoming"}],"user_woman":[{"name":"Anni"},{"name":"zhangsan"}]}'
json_data = json.loads(jsonstring)
print(json_data.get("user_man"))
print(json_data.get("user_woman"))
print(json_data.get("user_man")[0].get("name"))
print(json_data.get("user_woman")[1].get("name"))
```
也可以这样解析：
```
print(json_data["user_man"])
print(json_data["user_woman"])
print(json_data["user_man"][0]["name"])
print(json_data["user_woman"][1]["name"])
```

## 6.5 百度翻译API调用
1. 在 http://api.fanyi.baidu.com/api/trans/product/index 找到  百度翻译通用API 
2. 查阅技术文档：
接入方式：
通用翻译API HTTP地址：
http://api.fanyi.baidu.com/api/trans/vip/translate
通用翻译API HTTPS地址：
https://fanyi-api.baidu.com/api/trans/vip/translate

字段名 | 类型 | 必填参数 | 描述 | 备注
------ | ---- | -------- | ---- | ----
q | TEXT | Y | 请求翻译query | UTF-8编码
from | TEXT | Y | 翻译源语言 | 语言列表(可设置为auto)
to | TEXT | Y | 译文语言 | 语言列表(不可设置为auto)
appid |INT | Y |APP ID |可在管理控制台查看
salt | INT | Y |随机数 | 任意位数
sign | TEXT | Y | 签名 | appid+q+salt+密钥 的MD5值


# 7 表单交互
## 7.1 POST 方法
```
import requests
params = {
    'key1':'value1',
    'key2':'value2',
    'key3':'value3'
    }
html = requests.post(url,data=params) #post方法
print(html.text)
```
## 7.2 查看网页源代码提交表单
以豆瓣网 https://www.douban.com/ 为例，进行表单交互：
1. 打开豆瓣网，定位到登录元素，利用Chrome浏览器进行“检查”，找到登录元素所在的位置；
2. 根据步骤（1）在网页源代码中找到表单的源代码信息；
```
<form id="lzform" name="lzform" method="post" action="https://www.douban.com/accounts/login">
    <fieldset>
        <legend>登录</legend>
        <input type="hidden" value="index_nav" name="source">
        <div class="item item-account">
            <input type="text" name="form_email" id="form_email" value="" class="inp" placeholder="邮箱 / 手机号" tabIndex="1">
        </div>
        <div class="item item-passwd">
            <input name="form_password" placeholder="密码" id="form_password" class="inp" type="password" tabIndex="2">
            <div class="opt">
                <a href="https://www.douban.com/accounts/resetpassword">帮助</a>
            </div>
        </div>
        <div class="item item-submit">
            <input value="登录豆瓣" type="submit" class="bn-submit" tabIndex="4">
            <a href="/accounts/register" class="lnk-reg">注册帐号</a>
        </div>
        <div class="item-action">
            <label for="form_remember">
                <input name="remember" type="checkbox" id="form_remember" tabIndex="4">记住我
            </label>
            <ul class="item-action-third">
                <li><a class="wechat" href="https://www.douban.com/accounts/connect/wechat/?from=douban-web-anony-home" target="_blank" title="微信登录">微信登录</a></li>
                <li><a class="weibo" href="https://www.douban.com/accounts/connect/sina_weibo/?from=douban-web-anony-home" target="_blank" title="微博登录">微博登录</a></li>
            </ul>
        </div>
    </fieldset>
</form>
```
3. 对于表单源代码，有几个重要组成部分，分别是form标签的action属性和input标签； action属性为表单提交的URL；而input为表单提交的字段，input标签的name属性就是提交表单的字段名称；
4. 根据表单源代码，就可以构造表单进行登录网页了；
```
import requests
url = 'https://www.douban.com/accounts/login'
params = {
    'source':'index_nav',
    'form_email':'xxxx',
    'form_password':'xxxx'
    }
html = requests.post(url,params)
print(html.text)
```
> form_email和form_password字段为用户注册的账号和密码信息。

5. 通过比较未登录和登录豆瓣网页，可以看出登入后的网页右上角有账户名称；

## 7.3 逆向工程提交表单
通过Chrome浏览器的开发者工具中的Network选项卡查看表单交互情况，以此构造提交表单的字段信息， 同样以豆瓣网为例：
1. 进入豆瓣网，打开Chrome浏览器的开发者工具，选择Network选项；
2. 手工输入账号和密码后进行登录，此时会发现Network中会加载许多文件；
3. 打开第一个文件，可以看到请求的网址为表单源代码中的action属性值，请求方法为POST方法，接着往下查看， 在最底部的Form Data中就是提交的表单信息；


# 8 cookie模拟登录
Cookie，指某些网站为了辨别用户身份、进行session跟踪而储存在用户本地终端上的数据；<br>
互联网购物公司通过追踪用户的Cookie信息，给用户提供相关兴趣的商品；<br>
因为Cookie保存了用户的信息，便可通过提交Cookie来模拟登录网站；<br>
<br>
以豆瓣网为例，查找Cookie信息并提交来模拟登录豆瓣网:
1. 进入豆瓣网，打开Chrome浏览器的开发者工具，选择Network选项;
2. 手工输入账号和密码进行登录， 此时会发现Network中会加载许多文件;
3. 在新增的加载项中找到 www.douban.com，在 Headers 的标签下的 Request Headers 中可以找到 Cookie;
4. 在请求头中加入 cookie 信息即可完成豆瓣网的模拟登录；
```
# coding: utf-8
import requests
url = 'https://www.douban.com/'
header = {
    'Cookie': 'll="118111"; \
            bid=yjIoEhf3Im8; \
            _vwo_uuid_v2=D5A0B3828296A568176700D04903DE0D0|375fb62355fa2cbd3ad6342cf341c65d; \
            ps=y; push_noty_num=0; \
            push_doumail_num=0; ap=1; \
            _pk_id.100001.8cb4=4b0ba4b9-9b34-46de-8ae5-4e9921f3fd5a.1529931152.2.1529934907.1529931206.; \
            _pk_ses.100001.8cb4=*; \
            __utma=30149280.119509001.1523798658.1529931156.1529933907.3; \
            __utmb=30149280.4.9.1529934936332; \
            __utmc=30149280; \
            __utmz=30149280.1523798658.1.1.utmcsr=baidu|utmccn=(organic)|utmcmd=organic; \
            __utmv=30149280.16960; \
            _ga=GA1.2.119509001.1523798658; \
            _gid=GA1.2.1822887828.1529931193; \
            _gat_UA-7019765-1=1; \
            dbcl2="169608299:+1jYET3wDI0"'
}
req = requests.get(url, headers=header)
print(req.text)
```