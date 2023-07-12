---
title: 利用 shell 监控磁盘的读写情况
date: 2023-07-08
tags: ['shell','磁盘','磁盘读写'] 
---
## 背景
在做监控的时候，zabbix 自带的不支持做读写性能的监控，那么在有这个需求的时候就很蛋疼了，因此采用 shell 脚本的形式，然后将数据返回给 zabbix 中去

## 配置方式
###  conf 配置文件
```vim
UserParameter=qunhe.it.disk.info.iospeed[*],/etc/zabbix/zabbix_agentd.d/zabbix_disk_speed.sh $1
```
### shell

```bash
#!/bin/bash

DISK="$1"
DISK_MOUNT=""
DISK_SPEED=""
DISK_SPEED_UNIT=""
DISK_AVAIL=""
OUTPUT_FILE="/tmp/${DISK}_iotest"

if [ ! -d $DISK ] || [ -z $DISK ]; then
    echo "the disk isn't exist !"
    exit 1
fi

DISK_AVAIL="$(df $DISK | grep -v "Filesystem" | awk '{print $4}')"
DISK_AVAIL="$(expr $DISK_AVAIL / 1024 / 1024)"

if [ $DISK_AVAIL -lt 1 ]; then
    echo "disk is full!"
    exit 1
fi

DISK_MOUNT="$(df $DISK | grep -v 'Filesystem' | cut -d ' ' -f 1)"
sudo dd if=$DISK_MOUNT of=$DISK/test.iso bs=1024k count=500 conv=fdatasync iflag=direct >/dev/null 2>$OUTPUT_FILE
DISK_SPEED="$(cat $OUTPUT_FILE | tail -n 1 | cut -d ' ' -f 8)"
DISK_SPEED_UNIT="$(cat $OUTPUT_FILE | tail -n 1 | cut -d ' ' -f 9)"

if [[ $DISK_SPEED_UNIT = 'GB/s' ]]; then
    DISK_SPEED="$(expr $DISK_SPEED \* 1024)"
elif [[ $DISK_SPEED_UNIT = 'kB/s' ]]; then
    DISK_SPEED="$(expr $DISK_SPEED / 1024)"
fi

sudo rm $OUTPUT_FILE -rf
sudo rm /$DISK/test.iso -rf
echo $DISK_SPEED
```
## 使用方式
```shell
sh iospeed.sh /home
```