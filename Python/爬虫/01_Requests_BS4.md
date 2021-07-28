[toc]
# 1 爬虫原理和网页构造

## 1.1 爬虫原理

- 网络连接：
  - 计算机 带着 请求头和消息体 想服务器 发起一次Request请求，
  - 相应的服务器 会返回 本计算机相应的 HTML文件 作为Response。


- 爬虫原理：
  - 模拟计算机对服务器发起Request请求；
  - 接收服务器端的Response内容并解析、 提取所需的信息。

## 1.2 网页爬虫流程
### 1.2.1 多网页爬虫流程
有的网页存在多页的情况，每页的网页结构都相同或类似，这种类型的网页爬虫流程为：
1. 手动翻页并观察各网页URL构成特点，构造出所有页面的URL存入列表中；
2. 根据URL列表以此循环取出URL；
3. 定义爬虫函数；
4. 循环调用爬虫函数，存储数据；
5. 循环完毕，结束爬虫程序。

### 1.2.2 跨网页爬虫流程
分列表页和详细页，详细页与多网页爬虫一致；<br>
这种跨网页爬虫程序流程为：
1. 定义爬取函数，爬取列表页所有专题的URL；
2. 将专题URL存入列表中(种子URL)；
3. 定义爬取详细页数据函数；
4. 进入专题详细页爬取详细页数据；
5. 存储数据，循环完毕，结束爬虫程序。

## 1.3 网页构造
HTML + CSS + JavaScript


# 2 爬虫三大库
- Requests
- Lxml
- BeautifulSoup4

## 2.1 Requests库
### 2.1.1 requests.get()方法

```
# coding:utf-8
# python3.6.5
# centos7

import requests

head = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36'}

a = requests.get('http://www.baidu.com', headers=head)
print(a)   # 打印状态码
a.encoding = 'utf-8'
print(a.text)
```

### 2.1.2 requests.post()方法
post()方法用于提交表单来爬取需要登录才能获得数据的网站；

### 2.1.3 requests的异常
来源 | 异常名 | 原因 
:--- | :----- | :---
Requests | ConnectionError | 网络问题（如DNS查询失败、 拒绝连接等）
Response.raise_for_status() | HTTPError | HTTP请求返回了不成功的状态码（如网页不存在， 返回404错误）
Requests | Timeout | 请求超时
Requests | TooManyRedirects | 请求超过了设定的最大重定向次数

所有Requests显示抛出的异常都继承自 requests.exceptions.RequestsException。


## 2.2 BeautifulSoup 库
使用BeautifulSoup库可以解析Requests库请求的网页，并把网页源代码解析为Soup文档，以便于过滤提取数据；
```
# coding:utf-8

import requests
from bs4 import BeautifulSoup

head = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36'}

req = requests.get('http://www.baidu.com', headers=head)
print(req)   # 打印状态码
req.encoding = 'utf-8' # 设置返回的html的编码
# print(req.text)

soup = BeautifulSoup(req.text, 'html.parser')
print(soup.prettify())
```

### 2.2.1 BeautifulSoup解析器
解析器 | 使用方法 | 优点 | 缺点
:----- | :------- | :--- | :---
Python标准库 | BeautifulSoup(markup, 'html.parser') | Python内置标准库<br>执行速度适中<br>文档容错能力强 | Python2.7.3 or Python3.2.2 前的版本中，文档容错能力差
lxml HTML解析器 | BeautifulSoup(markup, 'lxml') | 速度快<br>文档容错能力强 | 需要安装C语言库
lxml XML解析器 | BeautifulSoup(markup, ['lxml', 'xml'])<br>BeautifulSoup(markup, 'xml') | 速度快<br>唯一支持XML的解析器 | 需要安装C语言库
html5lib |  BeautifulSoup(markup, 'html5lib') | 最好的容错性<br>以浏览器的方式解析文档<br>生成HTML5格式的文档<br>不依赖外部扩展 | 速度慢

### 2.2.2 find()、find_all()和selector()
解析得到的Soup文档，可以使用find()和find_all()方法以及selector()方法定位需要的元素；<br>
find()和find_all()两个方法用法如下：
```
find_all(tag, attibutes, recursive, text, limit, keywords)
find(tag, attibutes, recursive, text, keywords)
```

#### 1 find_all() 方法
```
soup.find_all('div', "item")    #查找div标签， class="item"
soup.find_all('div', class='item')
soup.find_all('div', attrs={"class": "item"})    # attrs参数定义一个字典
```

#### 2 find() 方法
find()方法与find_all()方法类似， <br>
只是find_all()方法返回的是文档中符合条件的所有tag，是一个集合，<br>
find()方法只返回一个tag；

#### 3 selector() 方法
```
soup.selector(div.item > a > h1)   #括号内容通过Chrome复制得到
```
该方法类似于中国>湖南省>长沙市， 从大到小，提取需要的信息，这种方式可以通过Chrome复制得到：
- 鼠标定位到想要提取的数据位置， 右击， 在弹出的快捷菜单中选择“检查”命令；
- 在网页源代码中右击所选元素；
- 在弹出的快捷菜单中选择 Copy -> selector。 这时便能得到：
```
#partB > div > div:nth-child(1) > div > div.bd > ul > li:nth-child(2) > div > p > span
```
此时，通过代码既可以得到房子的价格：
```
# coding: utf-8
import requests
from bs4 import BeautifulSoup

if __name__ == '__main__':
    url = 'http://www.ziroom.com/'
    header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/55.0.2883.87 \
                        UBrowser/6.2.3964.2 \
                        Safari/537.36'}

    req = requests.get(url, headers=header)

    soup = BeautifulSoup(req.text, 'lxml')
    # print(soup.prettify())
    prices = soup.select('#partB > div > div:nth-of-type(1) > div > div.bd > ul > li:nth-of-type(1) > div > p > span')
    print(prices)
```
结果会在屏幕上打印：[\<span>¥2990\</span>] 标签；

> 注意：div:nth-child(1) 和 li:nth-child(2) 在python运行中会报错，需要改成 div:nth-of-type(1) 和 li:nth-of-type(2)

此时的 li:nth-of-type(2) 代表一个价格，为了做平均价格分析，需要把所有的价格全部提取出来，把selector改为：
```
# 自如友家 南朝向的 全部价格
#partB > div > div:nth-of-type(1) > div > div.bd > ul > li > div > p > span

# 自如友家 所有朝向中的 全部价格
#partB > div > div > div > div.bd > ul > li > div > p > span
```

#### 4 get_text() 方法
对于select得到的prices列表，可以对其中的每个元素使用 get_text() 方法，获取标签间的文字；
```
    prices = soup.select('#partB > div > div > div > div.bd > ul > li > div > p > span')
    print(prices)
    for price in prices:
        print(price.get_text())
-----------------------------
[<span>¥2990</span>, <span>¥2030</span>, <span>¥2190</span>, <span>¥2490</span>, <span>¥3990</span>, <span>¥3260</span>]
¥2990
¥2030
¥2190
¥2490
¥3990
¥3260
```

## 2.3 Lxml 库
Lxml 库是基于 libxml2 这一个XML解析库的Python封装；<br>
该模块使用C语言编写，解析速度比BeautifulSoup更快；

