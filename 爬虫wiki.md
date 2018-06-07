### 利用python爬取wikiATC编码数据
```python
import requests
from bs4 import BeautifulSoup
import pandas as pd

#所要爬取的网站是https://zh.wikipedia.org/wiki/ATC代码_(),括号里面是对应的ATC编码，从A01到V20
#使用requests和beautifulsoup获取网页信息，爬wiki的时候要使用代理
#爬出的数据写成DataFrame格式后写入csv

def data_from_wiki():
  for x in ['A','B','C','D','QI','J','QJ','L','N','QN','P','QP','S','V']:
    for y in ['01','02','03','04','05','06','07','08','09','10','11','12','13',
              '14','15','16','20','51','54','52','53']:
        url='https://zh.wikipedia.org/wiki/ATC代码_('+x+y+')'
        print(url)
        try:
          html=requests.get(url,proxies=dict(https='socks5://Port:IP'))
        except:
          print('链接异常')
        else:
          soup=BeautifulSoup(html.text,'html.parser')
          data1=soup.select('.mw-headline')
          data2=soup.select('dd')
          list1=[]
          list2=[]
          for a in data1:
            list1.append(a.text.split('',1))
          for b in data2:
            list2.append(b.text.split('',1))
          DF1=pd.DataFrame(list1)
          DF2=pd.DataFrame(list2)
          DF1.to_csv('DATA1.csv',header=False,index=False,mode='a+')
          DF2.to_csv('DATA2.csv',header=False,index=False,mode='a+')

if __name__=='__main__':
  data_from_wiki()
```
