---
title: OpenWrt多拨使用教程  
date: 2023-02-28  
tags: ['openwrt','多拨']  
---
## 多拨类
多拨相关的插件主要是 **多线多拨** 和 **负载均衡** 插件。
<!-- more -->
### Syncdial **多线多拨**

使用macvlan驱动创建多个虚拟WAN口，支持并发多拨
```bash
opkg install luci-app-syncdial
```

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202302281925038.png"/>

### MWAN3负载均衡

支持多根网线或者多个PPPOE账号的同时拨号使用和负载均衡。并且还可以通过Ping方式来检测中断线路并自动屏蔽中断线路
```bash
opkg install luci-i18n-mwan3-zh-cn
```
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202302281926379.png"/>

## 其他

如发现无法安装或更新，请执行以下操作
### 更新OPKG软件列表
```bash
opkg update
```