---
title: 解决Nginx SSL 代理 Tomcat 获取 Scheme 总是 Http 问题
date: 2023-05-18
tags: ['nginx','tomcat','SSL'] 
---

## 背景
公司之前用的是http，但是出于苹果app审核和服务器安全性问题，要改为https，我们公司用的是沃通的ssl，按照沃通的官方文档提供的步骤完成服务器的配置。 架构上使用了 Nginx +tomcat 集群, 且nginx下配置了SSL,tomcat 没有配置SSL,项目使用https协议。

## 原因
![](img/解决Nginx%20SSL%20代理%20Tomcat%20获取%20Scheme%20总是%20Http%20问题/img1.png)
配置成功后明明是https url请求,发现 log里面，tomcat获取scheme的时候，一直是http，而不是想像中的https
```shell
0415 16:01:10 INFO  (PaymentInterceptor.java:44) preHandle() - requestStringForLog:    {
        "request.getRequestURL():": "http://m.xxx.com/payment/paymentChannel?id=212&s=a84485e0985afe97fffd7fd7741c93851d83a4f6",
        "request.getMethod:": "GET",
        "_parameterMap":         {
            "id": ["212"],
            "s": ["a84485e0985afe97fffd7fd7741c93851d83a4f6"]
        }
    }
```
request.getRequestURL() 输出出来的 一直是  
http://m.xxx.com/payment/paymentChannel?id=212&s=a84485e0985afe97fffd7fd7741c93851d83a4f6   
但是浏览器中的URL却是  
https://m.xxx.com/payment/paymentChannel?id=212&s=a84485e0985afe97fffd7fd7741c93851d83a4f6
下面我们进一步研究发现，java API上写得很清楚:  
```java
getRequestURL():
Reconstructs the URL the client used to make the request. 

The returned URL contains a protocol, server name, port number, and server path, 
but it does not include query string parameters.
```
![](img/解决Nginx%20SSL%20代理%20Tomcat%20获取%20Scheme%20总是%20Http%20问题/img2.png)

## 解决
### 配置nginx
放到`localhost /`里面去
```nginx
proxy_connect_timeout 300s;
proxy_send_timeout 900;
proxy_read_timeout 900;
proxy_buffer_size 32k;
proxy_buffers 4 64k;
proxy_busy_buffers_size 128k;
proxy_redirect off;
proxy_hide_header Vary;
proxy_set_header Accept-Encoding '';
proxy_set_header Referer $http_referer;
proxy_set_header Cookie $http_cookie;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto  $scheme;
```
> 其中的`proxy_set_header X-Forwarded-Proto $scheme;`起到了关键性的作用。

### 配置tomcat
```tomcat
<Valve className="org.apache.catalina.valves.RemoteIpValve"
remoteIpHeader="X-Forwarded-For"
protocolHeader="X-Forwarded-Proto"
protocolHeaderHttpsValue="https"/>
```
增加到`Engine`标签中  
配置双方的 X-Forwarded-Proto 就是为了正确地识别实际用户发出的协议是 http 还是 https。