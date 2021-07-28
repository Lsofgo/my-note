```
# coding: utf-8
import requests
from bs4 import BeautifulSoup
import time

header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                        AppleWebKit/537.36 (KHTML, like Gecko) \
                        Chrome/55.0.2883.87 \
                        UBrowser/6.2.3964.2 \
                        Safari/537.36'}

if __name__ == '__main__':
    urls = ['http://www.kugou.com/yy/rank/home/' + str(page_num) + '-8888.html?from=rank' for page_num in range(1, 2)]
    for url in urls:
        req = requests.get(url, headers=header)
        print(req)
        soup = BeautifulSoup(req.text, 'lxml')
        # print(soup.prettify())
        ids = soup.select('#rankWrap > div.pc_temp_songlist > ul > li > span.pc_temp_num')
        # print(ids[0].get_text().strip())
        titles = soup.select('#rankWrap > div.pc_temp_songlist > ul > li > a')
        # print(titles[0].get('title').strip())
        times = soup.select('#rankWrap > div.pc_temp_songlist > ul > li > span.pc_temp_tips_r > span')
        # print(times[0].get_text().strip())

        for i_d, title, voice_time in zip(ids, titles, times):
            data = {'id': i_d.get_text().strip(),
                    'singer': title.get('title').split('-')[0],
                    'song': title.get('title').split('-')[-1],
                    'voice_time': voice_time.get_text().strip(),
                    'link': title.get('href')
                    }
            print(data)
        time.sleep(1)
    
```