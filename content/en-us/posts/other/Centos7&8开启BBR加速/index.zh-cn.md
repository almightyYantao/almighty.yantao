---
title: CentOS 7 & 8 开启BBR加速
date: 2023-04-24
tags: ['centos','bbr','加速']  
---

## CentOS7 一键开启加速脚本
```bash
wget -N --no-check-certificate "https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

建议安装：BBR魔改加速  

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/img1.png"/>
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/img2.png"/>

## CentOS8 开启BBR加速
```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
reboot
```

## 检测是否开启成功
```bash
sysctl -n net.ipv4.tcp_congestion_control 。
## 会返回bbr 。
lsmod | grep bbr
## 会返回tcp_bbr
```