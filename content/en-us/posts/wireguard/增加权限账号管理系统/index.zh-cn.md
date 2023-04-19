---
title: 通过iptables进行wireguard的权限管理
date: 2023-04-18 
tags: ['wireguard','权限管理','权限']  
featured_image: img.webp
---

## 一、背景
由于目前openvpn的开源方案，链接VPN如果路由过多的话会导致链接速度变慢，效果非常的不理想，并且当iptables规则多的时候，转发明显性能下降；  
准备采用wireguard的方式来代替openvpn的隧道协议，但是wireguard目前没有一个很好的权限管理方案；

## 二、服务端

### 2.1、安装wireguard
```bash
curl -O https://raw.githubusercontent.com/atrandys/wireguard/master/wg_mult.sh && chmod +x wg_mult.sh && ./wg_mult.sh
```

### 2.2、设置访问权限
环境变量：`$WG_PEER_IP`可以获取连接着的IP地址
```bash
PostUp = /path/to/script.sh
PostDown = /path/to/teardown.sh
```
然后在根据情况设置ipset和iptables  

### 2.3、脚本案例
{{< accordion "/path/to/script.sh" >}}
```shell
#!/bin/bash
result=$(curl -s "VPN后端连接接口地址" | jq '.d')
permissionsAccept=($(jq '.accept[]' <<<$result))
permissionsDrop=($(jq '.drop[]' <<<$result))
  
# 创建新的ipset组
ipset create $WG_PEER_IP hash:ip
ipset create ${WG_PEER_IP}-drop hash:ip
for ((i=0;i<${#permissionsAccept[@]};i++))
do
  ipset add $WG_PEER_IP ${permissionsAccept[$i]//\"/}
done
 
for ((i=0;i<${#permissionsDrop[@]};i++))
do
  ipset add ${WG_PEER_IP}-drop ${permissionsDrop[$i]//\"/}
done
 
# 设置iptables生效规则
iptables -A FORWARD -s $WG_PEER_IP -m set --match-set ${WG_PEER_IP}-drop dst -j DROP
iptables -A FORWARD -s $WG_PEER_IP -m set --match-set $WG_PEER_IP dst -j ACCEPT
# 剩余没有匹配到的全部拒绝
iptables -A FORWARD -s $WG_PEER_IP -j DROP
 
if [ $code = -1 ];then
    exit 1 ;
fi
```
{{< /accordion >}}
{{< accordion "/path/to/teardown.sh" >}}
```shell
#!/bin/bash
iptables -D FORWARD -s $WG_PEER_IP -m set --match-set $WG_PEER_IP dst -j ACCEPT
iptables -D FORWARD -s $WG_PEER_IP -m set --match-set ${WG_PEER_IP}-drop dst -j DROP
iptables -D FORWARD -s $WG_PEER_IP -j DROP
ipset destroy $WG_PEER_IP
ipset destroy ${WG_PEER_IP}-drop
```
{{< /accordion >}}
### 2.4、常用脚本变量
`WIREGUARD_PUBKEY`: Wireguard 公钥。默认值为 $HOME/.wireguard/wg0.key.  
`WIREGUARD_PRIVATE_KEY`: Wireguard 私钥。默认值为 $HOME/.wireguard/wg0.pem.  
`WIREGUARD_SVC_IP`: Wireguard 服务 IP 地址。默认值为 127.0.0.1.  
`WIREGUARD_SVC_PORT`: Wireguard 服务端口。默认值为 9443.  
`WIREGUARD_DEV_IP`: 设备的 IP 地址。默认值为 127.0.0.1.  
`WIREGUARD_DEV_PORT`: 设备的端口。默认值为 9443.  
`WIREGUARD_NETWORK`: Wireguard 网络名称。默认值为 mywireguard.  
`WIREGUARD_CHANNEL`: Wireguard 通道名称。默认值为 channel0.  
`WIREGUARD_CONTROL_URL`: Wireguard 控制页面 URL。  
`WG_PEER_DNS`: Wireguard 通道的 peer DNS 名称。默认值为 mywireguard.com.  
`WG_PEER_IP`: Wireguard 通道的 peer IP 地址。默认值为 192.168.0.100.  
`WG_CHANNEL_FAILURE_ACTION`: Wireguard 通道失败时的处理方式。默认值为 continue.  
`WG_CONTROL_URL`: Wireguard 控制页面 URL。  
`WG_PRIVATE_KEY`: Wireguard 私钥。默认值为 $HOME/.wireguard/wg0.pem.  
`WG_PUBKEY`: Wireguard 公钥。默认值为 $HOME/.wireguard/wg0.key.  
`WG_DEV_IP`: 设备的 IP 地址。默认值为 127.0.0.1.  
`WG_DEV_PORT`: 设备的端口。默认值为 9443.  
`WG_NETWORK`: Wireguard 网络名称。默认值为 mywireguard.  
`WG_CHANNEL`: Wireguard 通道名称。默认值为 channel0.  
`WG_CONTROL_MODE`: Wireguard 控制模式 (0 表示禁用，1 表示启用)。默认值为 0.  

## 三、客户端

### 3.1、二进制文件
我对`wireguard-tools`的评价 {{< rating 5 5>}}  
{{< github-auto name="WireGuard/wireguard-tools" >}}
`mac`: https://www.123pan.com/s/cRk7Vv-NLSsH.html  提取码:2Zcc  
{{< notice warn >}}
这里要注意下，Mac因为他必须是sudo运行，但是不可能每次都输入密码，那么基本上等于废弃了，所以需要做一个初始化，把`wg-quick`设置成免输入密码的，代码如下：  
{{< /notice >}}
```bash
echo "$USER ALL=(ALL) NOPASSWD:这里改成你wg-quick的绝对路径" | sudo tee -a /etc/sudoers.d/wireguard
```
`win`: https://github.com/WireGuard/wireguard-tools/blob/master/src/wg-quick/darwin.bash  
{{< notice success >}}
win的可以直接使用这个脚本，也可以自己编译一个
{{< /notice >}}