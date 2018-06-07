### 使用itchat获取微信数据

```python
import pandas as pd
import itchat
itchat.login()
#会出现一个二维码，需要手机扫描登录

friends=itchat.get_friends()

def get_var(var):
    variable=[]
    for i in friends:
        variable.append(i[var])
    return variable

nickname=get_var('NickName')
sex=get_var('Sex')
province=get_var('Province')
city=get_var('City')
signature=get_var('Signature')

data={'nickname':nickname,'sex':sex,'province':province,'city':city,'signature':signature}
DF=pd.DataFrame(data)
DF.to_excel('wechatdata.xls',index=True)

```
