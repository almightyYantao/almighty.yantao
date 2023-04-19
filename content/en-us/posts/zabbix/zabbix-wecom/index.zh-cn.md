---
title: zabbix 警报推送至企业微信（图文版）
tags: ['zabbix','企业微信','报警']
index_img: "/img/zabbix.webp"
---

## 新增Python脚本
<!-- more -->
```python
# encoding: utf-8
import sys
import requests
import json
import os
import time
import re
 
url = 'http://10.1.1.59/zabbix/api_jsonrpc.php'
headers = {'Content-Type': 'application/json-rpc'}
graph_path = '/data/zabbix/images/'  # 定义图片存储路径
graph_url = 'http://10.1.1.59/zabbix/chart.php'  # 定义图表的url
loginurl = "http://10.1.1.59/zabbix/index.php"  # 定义登录的url
 
 
def uploadImg(path,accessToken):
    #img_url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/upload_media?key=" + key + "&type=file"
    img_url = "https://qyapi.weixin.qq.com/cgi-bin/media/uploadimg?access_token="+accessToken
    files = {'media': open(path, 'rb')}
    r = requests.post(img_url, files=files)
    re = json.loads(r.text)
    print(re)
    return re['url']
 
 
def get_itemid(message):
    #print(message)
    itemid = re.search(r'ITEMID:(\d+)', message).group(1)
    #itemid = 1
    return itemid
 
 
def get_imgUrl(itemid):
    session = requests.Session()
    try:
        loginheaders = {
            "Host": "10.1.1.59",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9"
        }
        # 定义请求消息头
 
        payload = {
            "name": 'yantao',
            "password": 'xxxxxxx',
            "autologin": "1",
            "enter": "Sign in",
        }
        # 定义传入的data
        login = session.post(url=loginurl, headers=loginheaders, data=payload)
        print(login)
        graph_params = {
            "from": "now-10m",
            "to": "now",
            "itemids": itemid,
            "width": "400",
        }
        # 定义获取图片的参数
        graph_req = session.get(url=graph_url, params=graph_params)
        # 发送get请求获取图片数据
        time_tag = time.strftime("%Y%m%d%H%M%S", time.localtime())
        graph_name = 'baojing_' + time_tag + '.png'
        # 用报警时间来作为图片名进行保存
        graph_name = os.path.join(graph_path, graph_name)
        # 使用绝对路径保存图片
        with open(graph_name, 'wb', ) as f:
            f.write(graph_req.content)
            # 将获取到的图片数据写入到文件中去
        return graph_name
    except Exception as e:
        print(e)
        return False
 
def getAccessToken():
    api_url = "https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=corpid&corpsecret=corpsecret"
    content = requests.get(api_url)
    #print(content.json())
    return content.json().get("access_token")
 
def getImgUrl(mediaId,accessToken):
    api_url = "https://qyapi.weixin.qq.com/cgi-bin/media/get?access_token="+accessToken+"&media_id="+mediaId
    content = requests.get(api_url)
    print(mediaId)
    print(content.json())
    return content.json().get("url")
 
def send_message(imgUrl,title,desc,openUrl,key):
    # 发送消息
    url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=" + key
    # message = title  # sys.argv[3]
    params = {
        "msgtype": "template_card",
        "template_card": {
            "card_type": "news_notice",
            "source": {
                "desc": "Zabbix网络警报",
                "desc_color": 0
            },
            "main_title":{
                "title":"Zabbix网络警报",
            },
            "quote_area":{
                "type":0,
                "quote_text":desc
            },
            "card_image": {
                "url": imgUrl
            },
            "card_action": {
                "type": 1,
                "url": openUrl,
                "appid": "APPID",
                "pagepath": "PAGEPATH"
            }
        }
    }
    req = requests.post(url, data=json.dumps(params))
    print(req.json())
 
 
if __name__ == '__main__':
    message = sys.argv[1]
    print(message)
    itemid = get_itemid(message)
    imgpath = get_imgUrl(itemid)
    accessToken = getAccessToken();
    imgUrl = uploadImg(imgpath,accessToken)
    #print(itemid)
    #print(imgpath)
    print(imgUrl)
    send_message(imgUrl,sys.argv[2],sys.argv[3],imgUrl,sys.argv[4])
    #accessToken = getAccessToken()
```

## 新增SH脚本

```shell
#!/bin/bash
echo $1 >> /data/zabbix/log.log
python /usr/lib/zabbix/alertscripts/wxcom.py $1 $2 $3 $4
```

把两个文件都放到这个目录下：/usr/lib/zabbix/alertscripts/

## 配置媒介
