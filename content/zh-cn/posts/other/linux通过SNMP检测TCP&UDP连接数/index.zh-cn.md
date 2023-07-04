---
title: linux通过SNMP检测TCP&UDP连接数  
date: 2023-03-11  
tags: ['zabbix','tcp','udp']  
featured_image: https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303112218722.png
---
## 先上图&介绍

记录一次全球化网络监控的看板建设；
之前都是通过zabbix来进行建设看板，但是用了一段时间后总是缺点感觉；后面通过大佬的介绍，试用了`Grafana`，这不试用还好，一试用，效果的展现让我无法自拔，这就是我想要的监控看板啊！能根据数值的不同进行颜色的区分，在大屏上一眼就可以看出当前哪一块网络出现了问题！

美中不足的是，上面的两列不能设置报警，报警貌似必须是图表形式的才可以；不过可以建立一个通用的报警看板，问题不大～
<!-- more -->
其他的一些监控项都是最简单基础的，我这边就不过多的赘述，大家可以自行上网搜索，或者之前引用之前`zabbix`的数据，主要讲解下`TCP连接数`和`UDP连接数`的获取

别的不说，这图还是很好看的；
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303112218722.png"/>

## 准备
- SNMP 
- 自定义OID

### snmp安装
```bash
yum install net-snmp
systemctl start snmpd
systemctl enable snmpd
```

### 修改配置文件
```bash {linenos=table,hl_lines=[4,5,7,8],title="/etc/snmp/snmpd.conf"}
# 修改团体名
com2sec notConfigUser default xxxxx

# view systemview included .1.3.6.1.2.1. （这两行注释）  
# view systemview included .1.3.6.1.2.1.25.1.1   
# 这两行新增，表示可以查看所有的信息，要不然只能查看上面那两个指定的信息
view all included .
view systemview included .1
```

## 创建获取连接数脚本
```bash {title="/etc/snmp/shell/zabbix.tcp.sh"}
#!/bin/bash
echo $2
echo integer
echo $(netstat -an|awk '/^tcp/ {++s[$NF]} END {for(a in s ) print a,s[a]}' | grep ESTABLISHED | awk '{print $2}')
```

这个是获取`TCP`连接数的脚本，如果要获取UDP的，请把`tcp`改成`udp`
设置脚本权限
```bash
chmod 766 zabbix.tcp.sh
```

## 修改SNMP配置文件

文件路径：`/etc/snmp/snmpd.conf`

```bash {title="/etc/snmp/snmpd.conf"}
pass .1.3.6.1.4.1.2021.21 /etc/snmp/shell/zabbix.tcp.sh
```

> 在最底下加入这条命令，后面的`shell`脚本就是上传创建的脚本路径

## 重启SNMP服务
```bash
service snmpd restart
```

## 后记

通过这种方式，我们就可以做更多的扩展，比如
- 获取连接人数
- 当前访问情况
- ...
