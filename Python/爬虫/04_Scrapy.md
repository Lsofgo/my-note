[toc]
# 9 Scrapy部署
## 9.1 安装Scrapy
Scrapy爬虫框架依赖许多第三方库，所以在安装Scrapy前，需确保以下第三方库均已安装：
- lxml
  - pip install lxml
- zope.interface
  - pip install zope.interface
- twisted
  - pip install Twisted[windows_platform]
- pyOpenSSL
  - pip install pyOpenSSL
- pywin32
  - pip install pywin32
  - 拷贝 Python35/Lib/site-packages/pywin32_system32/ 下的文件 到 C:\Windows\System32
- Scrapy
  - pip install scrapy

## 9.2 创建Scrapy项目
- 创建工程
```
cd Desktop/Scrapy/01_ZiRu
scrapy startproject ziru
```
- 执行结果如下：
```
New Scrapy project 'ziru', using template directory 'c:\\python27\\lib\\site-packages\\scrapy\\templates\\project', created in:
    C:\Users\Ray\Desktop\Scrapy\01_ZiRu\ziru

You can start your first spider with:
    cd ziru
    scrapy genspider example example.com
```
- 使用pycahrm打开创建的工程
- 在spiders下创建ziruspider.py

## 9.3 文件说明
```
Scrapy/01_ZiRu/                        
└── ziru                项目名
    ├── scrapy.cfg          scrapy项目的配置文件
    └── ziru                模块
        ├── __init__.py         
        ├── items.py            定义爬虫抓取的项目
        ├── middlewares.py      Spider的中间件
        ├── pipelines.py        主要作用为爬虫数据的处理
        ├── settings.py         主要作用是对爬虫项目的一些设置
        └── spiders
            ├── __init__.py
            └── ziruspider.py       编写爬虫代码
```
- scrapy.cfg
  - 定义默认设置文件的位置为xiaozhu模块下的settings文件；
  - 定义项目名称为ziru；
- items.py
  - 定义爬虫抓取的项目，就是定义爬取的字段信息；
- pipelines.py
  - 主要作用为爬虫数据的处理，主要用于爬虫数据的清洗和入库操作；
- settings.py
  - 主要作用是对爬虫项目的一些设置，如请求头的填写、设置pipelines.py处理爬虫数据等；

- spiders/ziruspider.py
  - 用于爬虫代码的编写；


## 9.4 写爬虫
### 9.4.1 一个最简单的栗子
```
import scrapy

class booksspider(scrapy.Spider):
    name = 'books'
    start_urls = ['http://books.toscrape.com/']

    def parse(self, response):
        for book in response.css('article.product_pod'):
            name = book.xpath('h3/a/@title').extract_first()
            price = book.css('p.price_color::text').extract_first()
            yield {
                'name': name,
                'price': price
            }
        next_url = response.css('ul.pager li.next a::attr(href)').extract_first()
        if next_url:
            next_url = response.urljoin(next_url)
            yield scrapy.Request(next_url, callback=self.parse)
```
- name属性：
  - 一个Scrapy项目中可能有多个爬虫，每个项目爬虫的name属性是其自身的唯一标识，在一个项目中不能有同名的爬虫；
- start_urls属性：
  - 一个爬虫总要从某个（或某些）页面开始爬取，这样的页面被称为起始爬取点，start_urls属性用来设置一个爬虫的起始爬取点；
- parse方法：
  - 当一个页面下载完成后，Scrapy引擎会回调一个我们指定的页面解析函数（默认是parse方法）解析页面；
  - 一个页面解析函数通常需要完成以下两个任务：
    - 提取页面中的数据（使用xpath或者css选择器）；
    - 提取页面中的链接，并产生对链接页面的下载请求；


页面解析函数（默认是parse）通常被实现成一个生成器函数， 每一项从页面中提取的数据以及每一个对链接页面的下载请求都由yield语句提交给Scrapy引擎。<br>
<br>

> 运行这个爬虫：
将运行结果保存到csv文件；

```
scrapy crawl books -o books.csv
```



### 9.4.2 稍复杂的栗子
#### 1 items.py
```
import scrapy

class ZiRuItem(scrapy.Item):
    title = scrapy.Field()
    address = scrapy.Field()
    price = scrapy.Field()
    house_type = scrapy.Field()
    floor = scrapy.Field()
```

#### 2 pipelines.py
```
class ZiruPipeline(object):
    def process_item(self, item, spider):
        fp = open('C:/Users/Ray/Desktop/ziru.txt', 'a+')
        fp.write(item['title']+'\n')
        fp.write(item['address'] + '\n')
        fp.write(item['price'] + '\n')
        fp.write(item['house_type'] + '\n')
        fp.write(item['floor'] + '\n')
        return item
```

#### 3 ziruspider.py
```
from scrapy.spiders import CrawlSpider
from scrapy.selector import Selector
from ziru.items import ZiRuItem

class ZiRu(CrawlSpider):
    name = 'ziru'
    start_urls = ['http://www.ziroom.com/z/vr/61279709.html']

    def parse(self, response):
        item = ZiRuItem()
        selector = Selector(response)
        title = selector.xpath('//div[@class="room_name"]/h2/text()').extract()[0].strip()
        address = selector.xpath('//div[@class="room_name"]/p/span[1]/text()').extract()[0].replace(" ", "").replace("\n", "")
        price = selector.xpath('//span[@id="room_price"]/text()').extract()[0].strip()
        house_type = selector.xpath('//div[@class="room_detail_right"]/ul[@class="detail_room"]/li[3]/text()').extract()[0].strip()
        floor = selector.xpath('//div[@class="room_detail_right"]/ul[@class="detail_room"]/li[4]/text()').extract()[0].strip()

        item['title'] = title
        item['address'] = address
        item['price'] = price
        item['house_type'] = house_type
        item['floor'] = floor

        yield item
```


#### 4 settings.py
```
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.87 UBrowser/6.2.3964.2 Safari/537.36'

ITEM_PIPELINES = {
   'ziru.pipelines.ZiruPipeline': 300,
}
```

## 9.5 运行Scrapy
进入到 项目 目录中，执行：
```
scrapy crawl ziru
```
可以看到：
```
2018-06-28 22:06:53 [scrapy.extensions.telnet] DEBUG: Telnet console listening on 127.0.0.1:6023
2018-06-28 22:06:53 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.ziroom.com/robots.txt> (referer: None)
2018-06-28 22:06:53 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.ziroom.com/z/vr/61279709.html> (referer: None)
2018-06-28 22:06:53 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.ziroom.com/z/vr/61279709.html>
{'address': '[朝阳望京]14号线东湖渠',
 'floor': '楼层： 1/21层',
 'house_type': '户型： 2室1厅',
 'price': '￥17000',
 'title': '东湖湾二期2居室'}
2018-06-28 22:06:53 [scrapy.core.engine] INFO: Closing spider (finished)
```
### 补充(windows环境)：
可以在 项目 目录下，写一个main.py的文件：
```
from scrapy import cmdline

cmdline.execute("scrapy crawl ziru".split())
```
这样直接运行 main.py就可以运行爬虫了；



# 10 编写Spider
## 10.1 Scrapy框架结构及工作原理
组件 | 描述 | 类型
---- | :--- | ----
engine | 引擎，框架的核心，其他所有组件在其控制下协同工作； | 内部组件
scheduler | 调度器，负责对spider提交的下载请求进行调度； | 内部组件
downloader | 下载器，负责下载页面（发送HTTP请求/接受HTTP响应） | 内部组件
spider | 爬虫，负责提取页面中的数据，产生对新页面的下载请求； | 用户实现
middleware | 中间件，负责对Request对象和Response对象进行处理； | 可选组件
item pipeline | 数据管道，负责对爬取到的数据进行处理； | 可选组件

对象 | 描述
---- | :---
request | Scrapy中的HTTP请求对象
response | Scrapy中的HTTP响应对象
item | 从页面中爬取的一项数据


> 上述对象在框架中的流动过程：

- 当SPIDER要爬取某URL地址的页面时，需使用该URL构造一个Request对象，提交给ENGINE；
- Request对象随后进入SCHEDULER按某种算法进行排队，之后的某个时刻SCHEDULER将其出队，送往DOWNLOADER；
- DOWNLOADER根据Request对象中的URL地址发送一次HTTP请求到网站服务器，之后用服务器返回的HTTP响应构造出一个Response对象，其中包含页面的HTML文本；
- Response对象最终会被递送给SPIDER的页面解析函数（构造Request对象时指定） 进行处理，页面解析函数从页面中提取数据，封装成Item后提交给ENGINE，Item之后被送往ITEMPIPELINES进行处理，最终可能由EXPORTER以某种数据格式写入文件（csv，json）；另一方面， 页面解析函数还从页面中提取链接（URL），构造出新的Request对象提交给ENGINE；


## 10.2 Request和Response对象
### 10.2.1 Request对象
Request对象用来描述一个HTTP请求；<br>
Request方法的参数列表：
```
Request(url[, callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority=0, dont_filter=False, errback])

    参数：
        url（必选）：
            请求页面的url地址，bytes或str类型；
            
        callback：
            页面解析函数，Callable类型，Request对象请求的页面下载完成后，由该参数指定的页面解析函数被调用；
            如果未传递该参数，默认调用Spider的parse方法；
            
        method：
            HTTP请求的方法，默认为 'Get'；
            
        headers：
            HTTP 请求的头部字典，dict类型；
            如果其中某项值为None，就表示不发送该HTTP头部；例如：{'Cookie': None}，禁止发送Cookie；
            
        body：
            HTTP请求的正文，bytes或str类型；
            
        cookies：
            Cookie信息字典，dict类型；
            
        meta：
            Request的元数据字典，dict类型，用于给框架中其他组件传递信息，比如中间件item pipeline；
            其他组件可以使用Request对象的meta属相访问该元数据字典（request.meta），也用于给响应处理函数传递信息；
            
        encoding：
            url和body参数的编码默认为'utf-8'；
            如果传入的url或body参数是str类型，就使用该参数进行编码；
            
        priority：
            请求的优先级默认值为0，优先级高的请求优先下载；
            
        dont_filter：
            默认情况下（dont_filter=False），对同一个url地址多次提交下载请求，后面的请求会被去重过滤（避免重复下载）；
            如果将该参数置为True，可以使请求避免被过滤，强制下载；例如：在多次爬取一个内容随时间变化的页面时（每次使用相同的URL），可以将该参数置为True；
            
        errback：
            请求出现异常或者出现HTTP错误时（如404页面不存在）的回调函数；
```

### 10.2.2 Response对象
Response对象用来描述一个HTTP响应，Response只是一个基类，根据响应内容的不同有如下子类：
 - TextResponse
 - HtmlResponse
 - XmlResponse

当一个页面下载完成时，下载器依据HTTP响应头部中的Content-Type信息创建某个Response的子类对象；<br>
我们通常爬取的网页，内容是HTML文本，创建的便是HtmlResponse对象，其中HtmlResponse和XmlResponse是TextResponse的子类；<br>
以HtmlResponse为例，HtmlResponse对象的属性及方法：
```
    url：
        HTTP响应的url地址，str类型；
        
    status：
        HTTP响应的状态码，int类型，例如200、404；
        
    headers：
        HTTP响应的头部，类字典类型，可以调用get或getlist方法对其进行访问；
        例如：
            response.headers.get('Content-Tyoe')
            response.headers.getlist('Set-Cookie')
            
    body：
        HTTP响应正文，bytes类型；
        
    text：
        文本形式的HTTP响应正文，str类型；
        由response.body使用response.encoding解码得到，即：
            response.text = response.body.decode(response.encoding)
            
    encoding：
        HTTP响应正文编码，它的值可能是从HTTP响应头部文件或者正文中解析出来的；
        
    request：
        产生该HTTP响应的Request对象；
        
    meta：
        即response.request.meta，在构造Request对象时，可将要传递给响应处理函数的信息通过meta参数传入； 响应处理函数处理响应时，通过response.meta将信息取出；
        
    selector：
        Selector对象用于在Response中提取数据；
        
    xpath：
        使用XPath选择器在Response中提取数据，实际上它是response.selector.xpath方法的快捷方式；
        
    css：
        使用CSS选择器在Response中提取数据，实际上它是response.selector.css方法的快捷方式；
        
    urljoin：
        用于构造绝对url；
        当传入的url参数是一个相对地址时， 根据response.url计算出相应的绝对url；
        例如：
            response.url为http://www.example.com/a， url为b/index.html；
            调用response.urljoin(url)的结果为http://www.example.com/a/b/index.html
```

## 10.3 Spider开发流程
实现一个Spider只需要完成下面4个步骤：
1. 继承scrapy.Spider；
2. 为Spider取名；
3. 设定起始爬取点；
4. 实现页面解析函数；


### 10.3.1 继承scrapy.Spider
Scrapy框架提供了一个Spider基类， 我们编写的Spider需要继承它：
```
import scrapy
class BooksSpider(scrapy.Spider):
    pass
```

这个Spider基类实现了以下内容：
- 供Scrapy引擎调用的接口，例如用来创建Spider实例的类方法from_crawler；
- 供用户使用的实用工具函数，例如可以调用log方法将调试信息输出到日志；
- 供用户访问的属性，例如可以通过settings属性访问配置文件中的配置；


### 10.3.2 为Spider命名
在一个Scrapy项目中可以实现多个Spider，每个Spider需要有一个能够区分彼此的唯一标识，Spider的类属性name便是这个唯一标识：
```
import scrapy
class BooksSpider(scrapy.Spider):
    name = 'books'
    pass
```
执行scrapy crawl命令时就用到了这个标识，告诉Scrapy使用哪个Spider进行爬取；


### 10.3.3 设定起始爬取点
Spider必然要从某个或某些页面开始爬取，我们称这些页面为起始爬取点， 可以通过类属性start_urls来设定起始爬取点：
```
import scrapy
class BooksSpider(scrapy.Spider):
    name = 'books'
    start_urls = ['http://books.toscrape.com/']
    pass
```

start_urls通常被实现成一个列表，其中放入所有起始爬取点的url（例子中只有一个起始点）；
- 对于起始爬取点的下载请求是由Scrapy引擎调用Spider对象的start_requests方法提交的；<br>    由于BooksSpider类没有实现start_requests方法，因此引擎会调用Spider基类的start_requests方法；
- 在start_requests方法中，self.start_urls便是我们定义的起始爬取点列表（通过实例访问类属性）， 对其进行迭代，用迭代出的每个url作为参数调用make_requests_from_url方法；
- 在make_requests_from_url方法中，我们找到了真正构造Reqeust对象的代码，仅使用url和dont_filter参数构造Request对象；
- 由于构造Request对象时并没有传递callback参数来指定页面解析函数，因此默认将parse方法作为页面解析函数；<br>此时BooksSpider必须实现parse方法，否则就会调用Spider基类的parse方法，从而抛出NotImplementedError异常（可以看作基类定义了一个抽象接口）；
- 起始爬取点可能有多个，start_requests方法需要返回一个可迭代对象（列表、生成器等），其中每一个元素是一个
Request对象；<br>这里，start_requests方法被实现成一个生成器函数（生成器对象是可迭代的），每次由yield语句返回一个Request对象；


由于起始爬取点的下载请求是由引擎调用Spider对象的start_requests方法产生的， 因此我们也可以在BooksSpider中实现start_requests方法（覆盖基类Spider的start_requests方法）， 直接构造并提交起始爬取点的Request对象。<br>
在某些场景下使用这种方式更加灵活，例如有时想为Request添加特定的HTTP请求头部，或想为Request指定特定的页面解析函数：
```
class BooksSpider(scrapy.Spider):
    # start_urls = ['http://books.toscrape.com/']
    def start_requests(self):
        yield scrapy.Request('http://books.toscrape.com/', 
            callback=self.parse_book, 
            headers={'UserAgent': 'Mozilla/5.0'}, 
            dont_filter=True)
            
    # 改用parse_book 作为回调函数
    def parse_book(response):
        pass
```

### 2.3.4 实现页面解析函数
页面解析函数也就是构造Request对象时通过callback参数指定的回调函数（或默认的parse方法）；<br>
页面解析函数是实现Spider中最核心的部分，它需要完成以下两项工作：
- 使用选择器提取页面中的数据，将数据封装后（Item或字典）提交给Scrapy引擎；
- 使用选择器或LinkExtractor提取页面中的链接，用其构造新的Request对象并提交给Scrapy引擎（下载链接页面）；
一个页面中可能包含多项数据以及多个链接， 因此页面解析函数被要求返回一个可迭代对象（通常被实现成一个生成器函数）， 每次迭代返回一项数据（Item或字典）或一个Request对象。


# 11 使用Selector提取数据

## 11.1 

