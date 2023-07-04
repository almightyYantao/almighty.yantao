---
title: curl 检测代理的可用性以及延迟  
date: 2023-02-20  
tags: ['curl','代理','检测','zabbix']  
---

## 背景
在办公网的代理翻墙的过程中，经常没办法第一时间知道代理失效了，因为我们自身不是高用的用户，每次挂了都需要员工来反馈，体感非常的不好，因此想着可以通过`zabbix`如果把当前的延迟、可用性检测起来
<!-- more -->
## 通过Curl 检测Google的延迟
这里为啥是curl而不是ping，因为默认ping事不支持代理的，然而curl也可以做到真正的是否可用

```bash
curl -o /dev/null -x socks5h://127.0.0.1:12126 -s --connect-timeout 5 -w %{time_starttransfer}"\n" $1
```

`socks5h`的地址需要改成你的代理地址
```
# 获取当前代理的可用性
```bash
url=$1
result=($(curl -x socks5h://localhost:12126 -I -s --connect-timeout 5 ${url} | head -1 | tr "\r" "\n"))
if [ "${result[1]}" == $2 -a "${result[2]}" == 'OK' ]
then
  echo 1
else
  echo 0
fi
```

## zabbix操作脚本
```bash
# 延迟
UserParameter=googleTime[*],/etc/zabbix/script/googleTime.sh $1

# 可用
UserParameter=googleCheck[*],/etc/zabbix/script/googleCheck.sh $1 $2
```