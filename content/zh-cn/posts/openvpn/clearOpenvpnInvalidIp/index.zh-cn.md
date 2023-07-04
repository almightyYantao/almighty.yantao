---
title: 清理自定义DHCP下发，异常断开导致IP地址占用的问题 
date: 2023-02-16  
tags: ['openvpn','dhcp']  
toc: true
---
## 背景

因为OpenVpn存在断开重连的机制，如果突然出现网络抖动，用户端没有触发断开命令，但是同时又触发了重新连接的命令，这时候VPN这里就会重新给下发一个新的地址，但是老的地址不会回收掉，这个也是很早之前DHCP被用完的真正原因
<!-- more -->
## 处理方式
在每一台openvpn的机器上新增一个shell脚本做定时任务，每天临晨2点开始循环遍历
```bash
#!/bin/bash
IFS=$'\n' # 修改默认分隔符
OLDIFS="$IFS"
 
# 清理无效的IP地址
function clearIp(){
    # 取出当前已经分配下去的所有IP地址
    result=$(curl -s "后端获取当前使用IP的接口地址");
    ipList=`echo $result | jq '.d.result'`;
    ipListLength=`echo $ipList | jq '.|length'`;
    date=`date "+%Y-%m-%d %H:%M:%S"`
 
    # 循环所有的数据
    for index in `seq 0 $ipListLength`
    do
        ip=`echo $ipList | jq -r ".[$index].ip"`;
        common_name=`echo $ipList | jq -r ".[$index].user"`
        # 拿到当前真正在使用的IP地址
        useIp=`cat /var/log/openvpn/status.log | grep CLIENT_LIST | awk '{if (NR>1){print $1}}' | cut -d ',' -f 4`
        # 如果当前IP已经不在使用了，那么就需要释放，防止后期地址池不够用的情况
        if [[ "${useIp[@]}"  =~ "${ip}" ]]; then
            echo "${date} | ${ip} | 当前IP属于使用状态，无需释放";
        else
            curl -s "释放IP的接口地址" >> /var/log/openvpn/disconnect.log
            echo "${date} | ${ip} | ${common_name} | 当前IP需要释放"
        fi
    done
}
 
# 清理Iptable的规则
function clearIptable(){
    iptableResult=($(iptables -L | grep 10.207 | grep ACCEPT | awk '{print $7}'))
    iptableResultCount=($(iptables -L | grep 10.207 | grep ACCEPT | awk '{print $7}' | wc -l))
    date=`date "+%Y-%m-%d %H:%M:%S"`
    for item in ${iptableResult[@]}
    do
        ip=`echo $item | cut -d '-' -f 2`
        common_name=`echo $item | cut -d '-' -f 1`
        ifconfig_pool_remote_ip=$ip
        useIp=`cat /var/log/openvpn/status.log | grep CLIENT_LIST | awk '{if (NR>1){print $1}}' | cut -d ',' -f 4`
        # 如果当前IP已经不在使用了，那么就需要释放，防止后期地址池不够用的情况
        if [[ "${useIp[@]}"  =~ "${ip}" ]]; then
            echo "${date} | ${ip} | 当前IP属于使用状态，无需释放";
        else
            echo "${date} | ${ifconfig_pool_remote_ip} | ${common_name} | 当前IP需要释放"
            /sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -m set --match-set ${common_name}-${ifconfig_pool_remote_ip} dst -j ACCEPT
            /sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -m set --match-set ${common_name}-${ifconfig_pool_remote_ip}-drop dst -j DROP
            /sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -j DROP
            /sbin/ipset destroy ${common_name}-${ifconfig_pool_remote_ip}
            /sbin/ipset destroy ${common_name}-${ifconfig_pool_remote_ip}-drop
        fi
    done
}
 
clearIp()
clearIptable()
```