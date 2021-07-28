```
# coding: utf-8

import requests
from lxml import etree

header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/55.0.2883.87 \
                        UBrowser/6.2.3964.2 \
                        Safari/537.36'}

if __name__ == '__main__':
    url = 'https://www.qiushibaike.com/text/'
    req = requests.get(url, headers=header)
    # print(req.text)

    html = etree.HTML(req.text)
    # selector = etree.tostring(html)
    # print(selector)
    id = html.xpath('//*[@id="qiushi_tag_120524981"]/div[1]/a[2]/h2/text()')
    print(id)
```