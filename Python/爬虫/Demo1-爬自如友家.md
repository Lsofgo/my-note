```
# coding: utf-8
import requests
from bs4 import BeautifulSoup

if __name__ == '__main__':
    base_url = 'http://www.ziroom.com/z/nl/z2.html?p=1'
    max_page_num = 50

    header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/55.0.2883.87 \
                        UBrowser/6.2.3964.2 \
                        Safari/537.36'}

    req = requests.get(base_url, headers=header)

    soup = BeautifulSoup(req.text, 'lxml')
    # print(soup.prettify())
    prices = soup.select('#houseList > li > div.txt > h3 > a')
    print(prices)
    for price in prices:
        print(price.get_text())
```