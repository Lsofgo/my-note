```
import json
import requests
import random
from hashlib import md5
from pprint import pprint


def get_sign(**kwargs):
    sign = kwargs['appid'] + kwargs['q'] + kwargs['salt'] + kwargs['key']
    return md5(sign.encode()).hexdigest()


if __name__ == '__main__':
    url = 'http://api.fanyi.baidu.com/api/trans/vip/translate'
    q = input('please input sentence in chinese:')
    get_random = str(random.randint(1000000000, 9999999999))
    par = {'q': q,
           'from': 'auto',
           'to': 'en',
           'appid': '20180621000178994',
           'salt': get_random,
           'key': 'WuycwRtng3tYnHG1jahC'}
    par['sign'] = get_sign(**par)
    # pprint(par)
    req = requests.get(url, par)
    result = json.loads(req.text)
    pprint(result.get('trans_result')[0].get('dst'))
```