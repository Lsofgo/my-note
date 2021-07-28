```
# coding: utf-8
import requests
from bs4 import BeautifulSoup
import time
import datetime

header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/55.0.2883.87 \
                        UBrowser/6.2.3964.2 \
                        Safari/537.36'}


# 解析目录页
def get_html(url):
    try:
        req = requests.get(url, headers=header)
        req.raise_for_status()
        req.encoding = req.apparent_encoding
    except Exception as e:
        print(e)
        return url
    return req


def get_index(html, base_url):
    soup = BeautifulSoup(html.text, 'lxml')
    # 爬书名
    book_name = soup.select('#info > h1')[0].get_text().strip()
    book_file = open('C:/Users/Ray/Desktop/' + book_name + '.txt', 'w', encoding='utf-8')

    # 爬章节名
    print(book_name)
    titles = soup.select('#list > dl > dd > a')

    book_size = len(titles)
    title_num = 1
    for title in titles:
        data = {'name': title.get_text().strip(),
                # 'url': base_url + title.get('href').strip(),
                'url': title.get('href').strip(),
                }
        print(data)
        print('%s\t%d/%d\t%s' % (str(datetime.datetime.now()), title_num, book_size, data.get('name')))
        title_num += 1
        book_file.write(data.get('name') + '\r\n')
        book_file.flush()

        # 爬网页遇错误码，重试
        html = get_html(data.get('url'))
        while html == data.get('url'):
            html = get_html(data.get('url'))
            print('try again...')
            time.sleep(1)

        # 爬正文
        txt = get_txt(html)
        book_file.write(txt + '\r\n')
        book_file.flush()
        time.sleep(1)

    book_file.close()


# 解析正文
def get_txt(html):
    soup = BeautifulSoup(html.text, 'lxml')
    # txt = soup.select('#content')
    txt = soup.select('#booktext')
    return txt[0].get_text()


if __name__ == '__main__':
    base_url = 'http://www.biquge.com.tw/'
    base_url = 'https://www.biquger.com/biquge/'
    book_id = '42893/'
    url = base_url+book_id

    html = get_html(url)
    while html == url:
        html = get_html(url)
        print('try again...')

    get_index(html, base_url)

```