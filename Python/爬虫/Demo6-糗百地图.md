```
# coding: utf-8
# 爬取糗事百科用户地址信息
# 1. 获取每个页面的url
# 2. 获取每个页面中的作者个人页面的url
# 3. 到每个作者的个人页面获取作者的居住地信息
# 4. 把居住地的省份信息取出
# 5. 统计省份信息
# 6. 将省份信息通过百度地图api，转换为经纬度
# 7. 将 省份,人数,经度，纬度 保存到csv
# 8. 把csv上传到BDP绘图

import requests
from lxml import etree
import json
import time
import csv
from pprint import pprint


# 1. 获取每个页面的url
def get_url(base_url, page_limit):
    page_url = []
    for page_num in range(1, page_limit + 1):
        url = base_url + '/hot/page/' + str(page_num)
        page_url.append(url)
    # pprint(page_url)
    return page_url


def get_html(url):
    # 访问间隔
    print(url)
    time.sleep(sleep_time)
    header = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) \
                            AppleWebKit/537.36 (KHTML, like Gecko) \
                            Chrome/55.0.2883.87 \
                            UBrowser/6.2.3964.2 \
                            Safari/537.36'}
    try:
        req = requests.get(url, header)
        return req
    except Exception as e:
        print('%s\n%s' % (e, url))

# 2. 获取每个页面中的作者个人页面的url
def get_user_url(base_url, *url_list):
    user_url_list = []
    for url in url_list:
        req = get_html(url)
        # pprint(html.text)
        selector = etree.HTML(req.text)
        user_urls = selector.xpath('//div[starts-with(@class,"article block untagged mb15")]')
        for user_info in user_urls:
            user_url = user_info.xpath('div[1]/a[1]/@href')
            if len(user_url) == 1:
                user_url_list.append(base_url + user_url[0])
            else:
                pass
    return list(set(user_url_list))

# 3. 到每个作者的个人页面获取作者的居住地信息
# 4. 把居住地的省份信息取出
# 5. 统计省份信息
def get_user_location(*user_url_list):
    address_info_list = []
    address_info = {}       # {省份: 该省份的作者数}
    for user_url in user_url_list:
        req = get_html(user_url)
        selector = etree.HTML(req.text)
        address = selector.xpath('/html/body/div[2]/div/div[3]/div[2]/ul/li[4]/text()')
        if len(address) > 0:
            if '·' in address[0]:
                print(address[0].split('·')[0])
                address_info_list.append(address[0].split('·')[0])
    set_address_info = set(address_info_list)
    for address in set_address_info:
        address_info[address.strip()] = [address_info_list.count(address)]
    return address_info


def get_longitude_and_latitude(**address_info):
    for address in address_info:
        time.sleep(1)
        if address == '国外':
            continue
        url = 'http://api.map.baidu.com/place/v2/search'
        par = {'query': address,
               'region': '全国',
               'output': 'json',
               'ak': 'pG4NFrIxIn34WeFESfGTf4qTUvnLAQAO'}
        req = requests.get(url, par)
        result = json.loads(req.text)
        location = result.get('results')[0].get('location')
        print(location)
        address_info[address].append(location['lng'])
        address_info[address].append(location['lat'])
    return address_info


def write_csv_file(file_path, **address_info):
    csv_file = open(file_path, 'w', encoding='utf-8')
    writer = csv.writer(csv_file)
    writer.writerow(['address', 'count', 'longitude', 'latitude'])
    for address in address_info:
        writer.writerow([address] + address_info[address])


if __name__ == '__main__':
    sleep_time = 1
    base_url = 'https://www.qiushibaike.com'
    csv_file_path = 'C:/Users/Ray/Desktop/address.csv'
    url_list = get_url(base_url, page_limit=13)
    user_url_list = get_user_url(base_url, *url_list)
    address_info = get_user_location(*user_url_list)
    get_longitude_and_latitude(**address_info)
    write_csv_file(csv_file_path, **address_info)

```