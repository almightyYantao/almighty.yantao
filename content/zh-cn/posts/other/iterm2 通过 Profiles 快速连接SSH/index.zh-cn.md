---
title: iterm2 通过 Profiles 快速连接SSH
date: 2023-07-27 
tags: ['iterm2','SSH','快速连接']  
---
## 背景
每次想要连接交换机或者其他SSH的时候都非常的麻烦，需要手动输入账号和密码。用过termius、warp、等总有点不如人意，最终还是拥抱了iterm2

## 开始教程
### 新增脚本
```shell
#!/usr/bin/expect

trap {
  set rows [stty rows]
  set cols [stty columns]
  stty rows $rows columns $cols < $spawn_out(slave,name)
} WINCH

set timeout 30
set host [lindex $argv 0]
set port [lindex $argv 1]
set user [lindex $argv 2]
set pswd [lindex $argv 3]
set param [lindex $argv 4]


if { "$param" == "ssh-rsa" } {
  spawn ssh -o "ServerAliveInterval=30" -o "StrictHostKeyChecking=no" -o "HostKeyAlgorithms=+ssh-rsa" -o "PubkeyAcceptedKeyTypes=+ssh-rsa" -p $port $user@$host
}

if { "$param" == "other" } {
  spawn ssh -o "ServerAliveInterval=30" -o "StrictHostKeyChecking=no" $param -p $port $user@$host
}

expect {
  "(yes/no)?" {send "yes\n";exp_continue;}
  -re "(p|P)ass(word|wd):" {send "$pswd\n"}
}

send "clear\r"

interact
```
这里总共设置了5个参数  
> 地址、端口、用户、密码、其他  
> 这里的其他主要是因为我们的交换机需要配置ssh-rsa，要不然连不上，你们自己需要的可以去改改
### 给权限
```shell
chmod +x /usr/local/bin/iterm2Login.sh
```
### 打开iterm2的配置
![](img/iterm2%20通过%20Profiles%20快速连接SSH/img1.png)
新增一个配置文件，在`Send text at start`中输入：
```shell
/usr/local/bin/iterm2Login.sh 地址 22 用户 密码 other
```
然后就可以直接在顶部的配置文件选项中直接连接啦~期间可以设置标签来进行分组设置，个人感觉还是非常不错的
![](img/iterm2%20通过%20Profiles%20快速连接SSH/img2.png)