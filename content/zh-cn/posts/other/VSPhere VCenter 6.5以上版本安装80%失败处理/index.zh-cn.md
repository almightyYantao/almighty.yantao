---
title: VSphere VCenter 6.5以上版本安装80%失败处理
date: 2023-05-19
tags: ['VSphere','VCenter']  
featured_image: https://raw.githubusercontent.com/almightyYantao/blog-img/master/202305191745679.png
---

## 起因
VCenter磁盘满，手贱ssh删除日志后重启失败，选择重装。故障：VC6.5安装卡80%，原因是安装包root密码过期了,root不可用。而且按照提示修改密码一直提示 Th...  

## 故障
VC6.5安装卡80%，原因是安装包root密码过期了,root不可用。而且按照提示修改密码一直提示 The password change operation failed.

## 解决
- 在80%时通过VSClient打开VC宿主虚机（注意要在第一时间打开，而不是已经提示错误。出现错误提示只能重新开始安装而不能继续）
- 重启在bios时按e进入GNU GRUB Menu
- 选择linux开头的这一行，在最后添加空格rw init=/bin/bash，然后F10进入
- 输入mount -o remount,rw / 回车（注意这个”/”别丢了）
- passwd 两次输入密码，建议用安装时设置的密码
- umount /（注意这个”/”别丢了）
- reboot -f

主要是因为系统版本的问题，网上分享的一般都是：`4602587` 版本的，这个版本有 BUG，可以换成 `U1G（8024368）`
