---
title: 制作一个内部的 zabbix-agent 快速部署脚本
tags: ['zabbix','zabbix-agent','agent']
date: 2023-07-06
featured_image: banner.png
---
## 下载官方的基础 agent 部署包
官方地址：[点击到达](https://www.zabbix.com/download_agents?version=5.0+LTS&release=5.0.36&os=Linux&os_version=3.0&hardware=i386&encryption=No+encryption&packaging=Archive&show_legacy=0)
![](img/制作一个内部的%20zabbix%20监控%20Agent/img1.png)
```sh
curl -O https://cdn.zabbix.com/zabbix/binaries/stable/5.0/5.0.36/zabbix_agent-5.0.36-linux-3.0-i386-static.tar.gz
```
## 编写 install 脚本
```shell
#!/bin/bash
## 变量定义
# 脚本所在路径
BASE_DIR=$(cd $(dirname $0);pwd)
# Zabbix_server连接IP
SERVER_IP=$1
# agent部署路径,默认/usr/local/zabbix_agent
INSTALL_DIR=$2
if [[ ! ${INSTALL_DIR} ]];then
  INSTALL_DIR=/usr/local/zabbix_agent
fi
if [[ ! -d ${INSTALL_DIR} ]];then
  mkdir -p ${INSTALL_DIR}/logs
fi
# agent部署包
INSTALL_PACK=$3
if [[ ! ${INSTALL_PACK} ]];then
  INSTALL_PACK=$(find ${BASE_DIR} -name "zabbix*.tar.gz")
fi
# agent监听端口，默认10050
AGENT_PORT=$4
if [[ ! ${AGENT_PORT} ]];then
  AGENT_PORT=10050
fi
## 环境监测
# 判断zabbix用户是否存在,不存在则创建
id zabbix &> /dev/null
if [[ $? != 0 ]];then
  useradd zabbix
fi
# 判断端口是否被占用
PORT_IF=$(ss -tanlu|grep -v 'Port'|grep "${AGENT_PORT}" | awk '{printf $5 "\n"}' | awk -F ':' '{printf $NF "\n"}' | sort | uniq)
if [[ ${PORT_IF} ]];then
  echo "端口 ${AGENT_PORT} 已被占用，退出安装"
  exit 1
fi
## 开始安装agent
# 解压安装包
tar -zxf ${INSTALL_PACK} -C ${INSTALL_DIR}
# 授权部署路径
chown -R zabbix:zabbix ${INSTALL_DIR}
if [ ! -d "${INSTALL_DIR}/conf/zabbix_agentd.conf.d" ];then
  mkdir ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
fi
# 修改配置文件
sed -i "s@Server=127.0.0.1@Server=${SERVER_IP}@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@# Include=/usr/local/etc/zabbix_agentd.conf.d/\*.conf@Include=${INSTALL_DIR}/conf/zabbix_agentd.conf.d/\*.conf@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@ServerActive=127.0.0.1@ServerActive=${SERVER_IP}@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@Hostname=Zabbix server@Hostname=${SERVER_IP}@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@# ListenPort=10050@ListenPort=${AGENT_PORT}@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@# PidFile=/tmp/zabbix_agentd.pid@PidFile=${INSTALL_DIR}/logs/zabbix_agent.pid@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@LogFile=/tmp/zabbix_agentd.log@LogFile=${INSTALL_DIR}/logs/zabbix_agentd.log@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
sed -i "s@# Timeout=3@Timeout=30@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf
 
# 复制自定义监控配置文件
# cp qunhe_it_disk.conf ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# cp qunhe_it_network.conf ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# cp zabbix_disk_speed.sh ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# cp zabbix_disk_discovery.sh ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# cp zabbix_network_tcp.sh ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# chmod 755 ${INSTALL_DIR}/conf/zabbix_agentd.conf.d/*.sh
 
# sed -i "s@/etc/zabbix/zabbix_agentd.d@${INSTALL_DIR}/conf/zabbix_agentd.conf.d@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf.d/qunhe_it_disk.conf
# sed -i "s@/etc/zabbix/zabbix_agentd.d@${INSTALL_DIR}/conf/zabbix_agentd.conf.d@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf.d/qunhe_it_network.conf
 
 
# 创建agent启动文件
cat > /usr/lib/systemd/system/zabbix_agentd.service << EOF
[Unit]
Description=Zabbix_agent service
After=syslog.target
After=network.target
 
[Service]
Type=simple
User=zabbix
Restart=always
KillMode=mixed
PIDFile=${INSTALL_DIR}/logs/zabbix_agent.pid
ExecStart=${INSTALL_DIR}/sbin/zabbix_agentd -c ${INSTALL_DIR}/conf/zabbix_agentd.conf
ExecStop=/bin/kill -SIGTERM $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=default.target
EOF
# 启动Zabbix_agent服务
systemctl daemon-reload
systemctl start zabbix_agentd.service
systemctl status zabbix_agentd.service &> /dev/null
if [[ $? = 0 ]];then
  echo "zabbix_agentd服务启动完成"
else
  echo "zabbix_agentd服务启动失败，请使用命令：systemctl status zabbix_agentd.service 查看失败原因"
fi
```
此时的目录下面是这样的结构
![](img/制作一个内部的%20zabbix%20监控%20Agent/img2.png)
在这里你可以编写自定义的监控脚本，毕竟默认的配置文件很多情况下无法满足内部情况
## 自定义监控案例：TCP 连接情况
### 新增 Shell 脚本
```shell
#!/bin/bash
/usr/sbin/ss -ant | awk '{++s[$1]} END {for(k in s) print k,s[k]}' | grep $1 | awk '{print $2}'
```
### 新增 conf 文件
```vim
UserParameter=qunhe.it.network.tcp[*],/usr/local/zabbix_agent/conf/zabbix_agentd.conf.d/zabbix_network_tcp.sh $1
```

将这两个文件同时放到你的打包目录下，此时的目录结构是：
![](img/制作一个内部的%20zabbix%20监控%20Agent/img3.png)
### 修改原来的 install 安装脚本
在自定义脚本的地方加入以下命令：
```shell
# 复制文件到安装目录下
cp zabbix_network_tcp.conf ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
cp zabbix_network_tcp.sh ${INSTALL_DIR}/conf/zabbix_agentd.conf.d
# 给 SH 赋权
chmod 755 ${INSTALL_DIR}/conf/zabbix_agentd.conf.d/*.sh
# 替换文件目录
sed -i "s@/etc/zabbix/zabbix_agentd.d@${INSTALL_DIR}/conf/zabbix_agentd.conf.d@g" ${INSTALL_DIR}/conf/zabbix_agentd.conf.d/qunhe_it_disk.conf
```
## 打包文件
```shell
tar -zcf install.tar.gz *
```
打包好了之后就可以去分发了～

## 在客户端安装 agent
```shell
# 安装
tar -zxf install.tar.gz
./install.sh 服务端IP  安装目录 自定义agent路径 端口号
```
- 服务端 IP：必填
- 安装目录：可选
- agent 路径：可选
- 端口号：可选
### 服务检查
```shell
service zabbix_agentd status
```
### 开启开机自启
```bash
systemctl enable zabbix_agentd
```
## 监控图表展示
![](img/制作一个内部的%20zabbix%20监控%20Agent/img4.png)