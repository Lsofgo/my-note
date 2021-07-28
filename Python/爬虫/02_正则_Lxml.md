[toc]
# 3 正则
## 3.1 正则表达式符号
字符 | 功能 | 示例
---- | ---- | ----
**普通字符** | |
literal | 匹配文本字符串的字面值 literal | foo
re1\|re2 | 匹配正则表达式 re1 或者 re2 | foo\|bar
. | 匹配任何字符（除了\n 之外） | b.b
^ | 匹配字符串起始部分 | ^Dear
$ | 匹配字符串终止部分 | /bin/*sh$
* | 匹配 0 次或者多次前面出现的正则表达式 | [A-Za-z0-9]*
+ | 匹配 1 次或者多次前面出现的正则表达式 | [a-z]+\\.com
? | 匹配 0 次或者 1 次前面出现的正则表达式 goo?
{N} | 匹配 N 次前面出现的正则表达式 | [0-9]{3}
{M,N} | 匹配 M～N 次前面出现的正则表达式 | [0-9]{5,9}
[…] | 匹配来自字符集的任意单一字符 | [aeiou]
[..x−y..] | 匹配 x～y 范围中的任意单一字符 | [0-9], [A-Za-z]
[^…] | 不匹配此字符集中出现的任何一个字符，包括某一范围的字符（如果在此字符集中出现） | [^aeiou], [^A-Za-z0-9]
(\*|+|?|{})? | 用于匹配上面频繁出现/重复出现符号的非贪婪版本 |（\*、 +、 ?、 {}） .*?[a-z]
(…) | 匹配封闭的正则表达式，然后另存为子组 | ([0-9]{3})?,f(oo|u)bar
**特殊字符** | | 
\d | 匹配一个数字，0-9
\D | 匹配一个非数字
\s | 匹配一个空白，即空格、tab键
\S | 匹配一个非空白
\w | 匹配一个单词字符，即a-z、A-Z、0-9、_
\W | 匹配一个非单词字符
\N | 匹配已保存的子组 N（参见上面的(…))
\c | 逐字匹配任何特殊字符 c（即，仅按照字面意义匹配，不匹配特殊含义） | \\., \\\\, \\*
\A(\Z) | 匹配字符串的起始（结束）（和^、$一样）
**扩展表示法** | |
不啦不啦不啦，不想写了，<br>反正用不到| |

## 3.2  正则方法
### 3.2.1 search()
search() 函数匹配并提取第一个符合规律的内容，返回一个正则表达式对象。<br>
search()) 函数的语法如下：
```
re.match(pattern, string, flags=0)
    pattern 为匹配的正则表达式；
    string  为要匹配的字符串
    flags   为标志位，用于控制正则表达式的匹配方式；如，是否区分大小写，多行匹配等；

import re
a = 'one1two2three3'
infos = re.search('\d', a)
print(infos)
print(infos.group())
```
可以使用 group() 方法来返回匹配到的字符串；

### 3.2.2 sub()
re模块提供了sub() 函数用于替换字符串中的匹配项， sub() 函数的语法如下：
```
re.sub(pattern, repl, string, count=0, flags=0)
    pattern 为匹配的正则表达式
    repl    为替换的字符串
    string  为要被查找的字符串
    counts  为模式匹配后替换的最大此时，默认0(表示替换所有的匹配)
    flags   为标志位，用于控制正则表达式的匹配方式；如，是否区分大小写，多行匹配等；

去除电话号码中的 - ：
import re
phone_num = '136-1101-1101'
new_phone = re.sub('\D', '', phone_num)
print(new_phone)
```
sub() 函数用途类似字符串中的 replace()；<br>
但是sub() 函数更加灵活，可以通过正则表达式来匹配需要替换的内容；


### 3.2.3 findall()
findall() 函数匹配所有符合规律的内容，并以列表的形式返回结果，例如：
```
import re
a = 'one1two2three3'
infos = re.search('\d', a)
print('search: %s' % infos.group())

infoss = re.findall('\d', a)
print('findall:')
print(infoss)
```

## 3.3 re模块的修饰符
修饰符 | 描述
:----- | :---
 re.I  | 使匹配对大小写不敏感
 re.L  | 做本地化识别(locale-aware)匹配
 re.M  | 多行匹配，影响 ^ 和 $
 re.S  | 是匹配包括换行在内的所有字符
 re.U  | 根据Unicode字符集解析字符，这个标志影响 \w、\W、\b、\B
 re.X  | 该标志通过给予更灵活的格式，以便将正则表达式写的更易理解

# 5 Lxml库与Xpath语法
Lxml库是基于libxml2的xml解析库的python封装；<br>
该模块使用C语言编写，解析速度比BeautifulSoup更快；<br>

## 5.1 lxml库使用
### 1 修正html代码
首先导入lxml中的etree库，然后利用etree.HTML进行初始化，最后把结果打印出来；<br>
etree库把HTML文档解析成Element对象，再使用tostring() 方法可以返回HTML页面内容；
```
from  lxml import etree
text = '''
<div>
    <ul>
        <li class='red'><h1>red flowers</h1></li>
        <li class='yellow'><h2>yellow flowers item</h2></li>
        <li class='white'><h3>white flowers</h3></li>
        <li class='black'><h4>black flowers</h4></li>
        <li class='blue'><h5>blue flowers</h5>
    </ul>
</div> 
'''
html = etree.HTML(text)
print(html)
result = etree.tostring(html)
print(result)
```
最终打印的结果里，lxml会自动补全最后一个 li 标签；

### 2 读取HTML文件
除了直接读取字符串，lxml还支持从文件中提取内容；
```
from lxml import etree
html = etree.parse('a.html')
result = etree.tostring(html)
print(result)
```
> 注意：该HTML文件只能在本地打开

### 3 解析HTML文件
利用requests库获取HTML文件，用lxml库来解析HTML文件；
```
import requests
from lxml import etree

header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/66.0.3359.181 \
                        Safari/537.36'}
url = 'http://www.baidu.com/'
req = requests.get(url, headers=header)
html = etree.HTML(req.text)
result = etree.tostring(html)
print(result)
```

## 5.2 Xpath语法
### 5.2.1 节点关系
```
<user_database>
    <user>
        <name>xiao ming</name>
        <sex>J K. Rowling</sex>
        <id>43</id>
        <goal>89</goal>
    </user>
</user_database>
```
#### 1 父节点

#### 2 子节点

#### 3 同胞节点

#### 4 先辈节点

#### 5 后代节点

### 5.2.2 节点选择
Xpath使用路径表达式在XML文档中选取节点；<br>
节点是通过沿着路径或者step来选取的；<br>

#### 节点选择
表达式 | 描述
:----- | :---
nodename | 选取次节点的所有子节点
/  | 从根节点选取
// | 从匹配选择的当前节点选择文档中的节点，而不考虑它们的位置
.  | 选取当前节点
.. | 选取当前节点的父节点
@  | 选取属性

#### 节点选择实例
路径表达式 | 结果
:--------- | :---
user_database   | 选取元素user_database的所有子节点
/user_database  | 选取根元素user_database<br>假如路径起始于正斜杠 / ，则此路径始终代表到某元素的绝对路径
user_database/user | 选取属于user_database的紫苑寺的所有user元素
//user | 选取所有user元素，而不管它们在文档中的位置
user_database//user | 选择属于user_database元素的后代的所有user元素，而不管它们位于user_database之下的什么位置
//@attribute | 选取名为 attribute 的所有属性

#### 谓语
Xpath语法中的谓语用来查找某个特定的节点或者包含某个指定值的节点，谓语被嵌在方括号中，常见的谓语如下：
路径表达式 | 结果
:--------- | :---
/user_database/user[1] | 选取属于user_database子元素的第一个user元素
//li[@attribute] | 选取所有拥有名为attritube属性的li元素
//li[@attritbue='red'] | 选取所有li元素，且这些元素拥有值为red的attribte属性

Xpath中也可以使用通配符来选取位置的元素，常用的就是 * 通配符，它可以匹配任何元素节点；

### 5.2.3 使用技巧
在爬虫实战中，Xpath路径可以通过Chrome复制得到：<br>
1. 鼠标光标定位到想要提取的数据位置， 右击， 从弹出的快捷菜单中选择“检查”命令；
2. 在网页源代码中右击所选元素；
3. 从弹出的快捷菜单中选择Copy Xpath命令。这时便能得到：
```
//*[@id="qiushi_tag_118732380"]/div[1]/a[2]/h2
```
举个栗子，爬取糗事百科作者id：
```
import requests
from lxml import etree

if __name__ == '__main__':
    url = 'https://www.qiushibaike.com/text/'
    req = requests.get(url, headers=header)
    # print(req.text)

    html = etree.HTML(req.text)
    # print(etree.tostring(html))
    id = html.xpath('//*[@id="qiushi_tag_120524981"]/div[1]/a[2]/h2/text()')
    print(id)
```
> 注意：可以通过在xpath后面加 /text() 来获取标签文字;<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
否则会爬出来 这么个鬼：<Element h2 at 0x2040fd2eb48>

如果要进行用户id的批量爬取时，通过类似 BeautifulSoup中删除谓语的部分是不行的；<br>
这样做：
```
xpath谓语是这样的：//*[@id="qiushi_tag_120524981"]/div[1]/a[2]/h2

谓语的前半截 id="qiushi_tag_120524981 是动态的，查看网页源码：
<div class="article block untagged mb15 typs_old" id='qiushi_tag_114983156'>

把 id..... 换成 class.....
先把这部分取下来，在对这部分的结果 爬 div[1]/a[2]/h2， 就可以得到全部的ID了

html = etree.HTML(req.text)
# print(etree.tostring(html))
divs = html.xpath('//div[@class="article block untagged mb15 typs_hot"]')
for div in divs:
    id = div.xpath('div[1]/a[2]/h2/text()')
    print(id)
```

### 5.2.4 特殊的语法姿势
#### 相同字符开头的多个标签
```
from lxml import etree
html1 = '''
<li class="tag-1">需要的内容1</li>
<li class="tag-2">需要的内容2</li>
<li class="tag-3">需要的内容3</li>
'''
selector = etree.HTML(html1)
contents = selector.xpath('//li[starts-with(@class,"tag")]/text()')
for content in contents:
print(content) # starts-with()可获取类似标签的信息
```

#### 标签嵌套标签
```
from lxml import etree
html2 = '''
<div class="red">需要的内容1
<h1>需要的内容2</h1>
</div>
'''
selector = etree.HTML(html2)
content1 = selector.xpath('//div[@class="red"]')[0]
content2 = content1.xpath('string(.)')
print(content2) # string(.)方法可用于标签套标签情况
```


