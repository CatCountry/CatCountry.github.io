---
title: docker-alpine-crontab
date: 2021-07-07 13:36:02
tags: ['k8s', 'crontab']
---
## Docker中alpine镜像crontab使用

### 背景
线上做数据迁移时，出现了一次Pod被驱逐的情况，导致迁移任务执行一半，需要手工重新执行。  
查看k8s的日志情况，发现是日志打印太多，导致机器内可用存储不足，所以pod被驱逐了。  
现有情况下，不能动态修改日志级别，并且不想修改重新打包。所以决定使用crontab，定时删除以前的日志。

### 增加crontab
```
# echo "find /data/data/logs/ -type f -mmin +30 -print -delete" > /data/clear-log.sh
# crontab -e
*/15    *       *       *       *       sh /data/clear-log.sh
```
### 问题
发现crontab没有执行，在网上看到下面的方案：  
`alpine内嵌的是BusyBox，使用alpine的crontab实际就是使用BusyBox的crond服务`
```sh
# crond
# ps | grep crond
```
任务正常开始执行