---
title: 通过 Haproxy 实现 Shadowshadows 负载均衡
description: HAProxy是一个使用C语言编写的自由及开放原始码软件[3]，其提供高可用性、负载均衡，以及基于TCP和HTTP的应用程序代理。
---

## 介绍

缺点：所有的SS的加密方式和密码必须一致
介绍：HAProxy是一个使用C语言编写的自由及开放原始码软件，其提供高可用性、负载均衡，以及基于TCP和HTTP的应用程序代理。
<!-- more -->
## 安装Haproxy

```shell
yum install haproxy
```

## 配置

```shell
global
    chroot  /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    user    haproxy
    group   haproxy
defaults
    mode    tcp                         #服务器默认的工作模式
    balance roundrobin                  #服务器默认使用的均衡模式
    retries 3                           #三次连接失败表示服务器不可用
    maxconn 5000                        #最大连接数
    timeout connect 500ms               #连接超时
    timeout client  3s                  #客户端超时
    timeout server  3s                  #服务器超时


listen WebPanel
    mode    http                        #这里使用HTTP模式
    bind    10.1.1.58:9595               #WEB服务端口
    stats   refresh 5s                  #自动刷新时间
    stats   uri  /                      #WEB管理地址
    stats   auth admin:xxxx         #账号密码
    stats   hide-version                #隐藏版本
    stats   admin if TRUE               #验证通过则赋予管理权

frontend shadowsocks-in
    bind *:50001
    default_backend servers

backend servers
    mode tcp
    balance roundrobin
    # option external-check
    option tcp-check
    # external-check command "/etc/haproxy/haproxy-shadowsocks-checker.py"

    server ss1 43.156.100.84:2195 check inter 500 rise 2 fall 4 weight 100   #SS/SSR服务器地址与端口
    server ss2 42.157.192.42:10108 check inter 500 rise 2 fall 4 weight 100
```

`server`：后面首先跟名字，名字随便起呗，自己能够区分就行。紧接着跟SS的公网IP+端口，端口也就是SS/SSR的端口。
`check`：是检测的意思，这段配置很重要
`inter`：单位毫秒，我配置的500，即500毫秒检测一次目标服务器。
`rise2`：设定健康状态检查中，某离线的服务器从离线状态转换至正常状态需要成功检查的次数，这里我设置的2次。
`fall4`：确认服务器从正常状态转换为不可用状态需要检查的次数，这里是4次。
`weight`：权重，值越大代表这台机器工作的机会越多，这里我们可以把一台线路较好的机器的权重设置高一些。
`balance`：负载均衡方式

- roundrobin：基于权重进行轮叫，在服务器的处理时间保持均匀分布时，这是最平衡、最公平的算法。此算法是动态的，这表示其权重可以在运行时进行调整，不过，在设计上，每个后端服务器仅能最多接受4128个连接；
- static-rr：基于权重进行轮叫，与roundrobin类似，但是为静态方法，在运行时调整其服务器权重不会生效；不过，其在后端服务器连接数上没有限制；
- leastconn：新的连接请求被派发至具有最少连接数目的后端服务器；在有着较长时间会话的场景中推荐使用此算法，如LDAP、SQL等，其并不太适用于较短会话的应用层协议，如HTTP；此算法是动态的，可以在运行时调整其权重；
- source：将请求的源地址进行hash运算，并由后端服务器的权重总数相除后派发至某匹配的服务器；这可以使得同一个客户端IP的请求始终被派发至某特定的服务器；不过，当服务器权重总数发生变化时，如某服务器宕机或添加了新的服务器，许多客户端的请求可能会被派发至与此前请求不同的服务器；常用于 负载均衡无cookie功能的基于TCP的协议；其默认为静态，不过也可以使用hash-type修改此特性；

## 修改Shadowsocks配置

```shell
{
    "server":"127.0.0.1", # 这里改成Haproxy的地址
    "server_port":50001, # 这里改成Haproxy监听的端口
    "local_port":1080, # 这里本地监听端口
    "password":"xxxxxxxx",
    "timeout":60,
    "method":"aes-256-gcm",
    "mode":"tcp_and_udp"
}
```
