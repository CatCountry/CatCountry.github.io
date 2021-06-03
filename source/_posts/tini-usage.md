---
title: tini usage
date: 2021-06-02 19:05:19
tags: ['k8s', 'tini']
categories: k8s
---

## Tini使用

#### 问题

k8s测试环境中以`OpenJDK:alpine`为基础镜像运行的SpringBoot项目中线程池不知道什么原因全部不工作了，想要用 jstck 查看线程状态。  
进入容器内
```shell
/ # ps -ef | grep java
    1 root      1h32 java -jar app.jar 
/ # jstack 1
1: Unable to get pid of LinuxThreads manager thread    
```

> PID 1介绍
> https://shareinto.github.io/2019/01/30/docker-init%281%29/
> https://coolshell.cn/articles/17998.html 
> https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/


#### 解决方案  
1. 基础镜像封装tini，并使用tini启动
```yaml
    containers:
    - name: my-app
        command: ["sh", "-c", "cd /data &&  tini -- java -jar app.jar"]
```
容器内部
```shell
/ # ps -ef 
PID   USER     TIME  COMMAND
    1 root      0:00 tini -- java -jar app.jar
    6 root      1h39 java -jar app.jar
```
2. 使用OpenJDK:8u222-jdk(非alpine版本)
```yaml
    containers:
    - name: org-api-run
        image: openjdk:8u222-jdk
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "cd /data && java -jar app.jar"]
```
容器内部
```bash
/# ps -ef
root         1     0  0 14:22 ?        00:00:00 sh -c cd /data && java -jar app.jar
root         6     1  1 14:22 ?        00:03:14 java -jar app.jar
```