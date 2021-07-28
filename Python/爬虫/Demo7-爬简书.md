```
# 1. 使用cookie登录 https://www.jianshu.com/
# 2. 爬取简书电影专题下10页页面
#    通过 Chrome -> F12 -> Network -> XHR 找到请求的异步加载的网页URL
#    https://www.jianshu.com/c/1hjajt?order_by=added_at&page=1
# 3. 使用多进程 将文章用户ID、文章发表日期、文章标题、文章内容、浏览量、评论数、点赞数、打赏数 爬出来，保存到MongoDB

# coding: utf-8
import os
import requests
from lxml import etree
import pymongo
from multiprocessing import Pool


def connect_mongodb():
    client = pymongo.MongoClient('192.168.23.128', 27017)
    jianshu = client['jianshu']
    jianshu_article = jianshu['jianshu_article']
    return jianshu_article


def make_page(page_url):
    page_url_list = []
    for page_num in range(1, 2):
        page_url_list.append(page_url + str(page_num))
    return page_url_list


def get_html(url):
    # 访问间隔
    print(os.getpid(), url)
    # time.sleep(sleep_time)
    header = {'User-Agent':
                  'Mozilla/5.0 (Windows NT 10.0; \WOW64) \
                  AppleWebKit/537.36 (KHTML, like Gecko) \
                  Chrome/55.0.2883.87 \
                  UBrowser/6.2.3964.2 \
                  Safari/537.36',
              'Cookie':
                  'read_mode=day; default_font=font2; \
                  remember_user_token=W1sxMjU0MjE1Nl0sIiQyYSQxMSQ1Y3pwRDVFZE40aFJhY2pXY01MTExP\IiwiMTUzMDEwOTk3OC4yNDQ2NzM3\
                  Il0%3D--153f3ac58e4b1d18833ed84fef33e6c0b951fc5f; \
                  Hm_lvt_0c0e9d9b1e7d617b3e6842e85b9fb068=1530109777,1530109876; \
                  Hm_lpvt_0c0e9d9b1e7d617b3e6842e85b9fb068=1530111449; \
                  sensorsdata2015jssdkcross=%7B%22distinct_id%22%3A%2212542156%22%2C%22%24device_id%22%3A%22162250f664a4ca-\
                  084286897c2467-4a541326-2073600-162250f664b46a%22%2C%22props%22%3A%7B%22%24latest_traffic_source_type%22%\
                  3A%22%E4%BB%98%E8%B4%B9%E5%B9%BF%E5%91%8A%E6%B5%81%E9%87%8F%22%2C%22%24latest_referrer%22%3A%22%22%2C%22%\
                  24latest_referrer_host%22%3A%22%22%2C%22%24latest_search_keyword%22%3A%22%E6%9C%AA%E5%8F%96%E5%88%B0%E5%8\
                  0%BC_%E7%9B%B4%E6%8E%A5%E6%89%93%E5%BC%80%22%2C%22%24latest_utm_source%22%3A%22desktop%22%2C%22%24latest_\
                  utm_medium%22%3A%22index-collections%22%7D%2C%22first_id%22%3A%22162250f664a4ca-084286897c2467-4a541326-2\
                  073600-162250f664b46a%22%7D; \
                  locale=zh-CN; \
                  _m7e_session=a1425f42c745e3eef5eb7483b0c83843'}
    try:
        req = requests.get(url, headers=header)
        return req
    except Exception as e:
        print('%s\n%s' % (e, url))


def get_article_url(base_url, *page_url_list):
    article_url_list = []
    for page_url in page_url_list:
        html = get_html(page_url)
        selector = etree.HTML(html.text)
        for article_url in selector.xpath('//li[starts-with(@id, "note-")]/a/@href'):
            article_url_list.append(base_url + article_url)
    return article_url_list


def get_article_info(collection, article_url):
    html = get_html(article_url)
    selector = etree.HTML(html.text)
    title = selector.xpath('//h1[@class="title"]/text()')
    # auther = selector.xpath('//span[@class="name"')
    print(title)





if __name__ == '__main__':
    base_url = 'https://www.jianshu.com'
    page_url = 'https://www.jianshu.com/c/1hjajt?order_by=added_at&page='
    p = Pool(2)
    collection = connect_mongodb()
    collection = 1
    page_url_list = make_page(page_url)
    article_url_list = get_article_url(base_url, *page_url_list)
    for article_url in article_url_list:
        p.apply_async(get_article_info, args=(collection, article_url))
    p.close()
    p.join()
```