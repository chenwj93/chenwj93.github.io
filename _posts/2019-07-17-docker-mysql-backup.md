---
layout: post
title: Docker mysql timing backup
subtitle: docker mysql 定时备份
date: 2019-07-29
categories: docker
cover: 
tags: all docker mysql
---

> 库存 docker-mysql

## 创建MySQL容器
#### 拉取镜像
>docker pull registry.cn-hangzhou.aliyuncs.com/acs-sample/mysql:5.7

#### 修改一下tag
>docker tag registry.cn-hangzhou.aliyuncs.com/acs-sample/mysql:5.7 mysql:5.7

#### 启动服务
>docker run --name="mysql5.7" -v /mysql/data:/var/lib/mysql -v /mysql/conf.d:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=password --restart=always -d -i -p 3316:3306 mysql:5.7

#### 进入容器
>docker exec -it mysql5.7 /bin/bash

#### 启动mysql客户端命令行
>mydql -h 127.0.0.1 -u root -p

## 创建定时任务

> 假定 设置/home/mysql/timingBackup文件夹为备份文件夹，并挂载到容器的相同文件夹下
---
#### 进入容器
> docker exec -it mysql /bin/bash

#### 进入timingBackup 文件夹
> cd /home/mysql/timingBackup

#### 创建mysqldump.sh执行文件
> vim mysqldump.sh

#### 编辑脚本
```
day=`date +%Y%m%d`
backpath=/home/mysql/timingBackup/$day
user=username
passwd=password
serverName=backupfilename

mkdir -p $backpath
mysqldump -h '127.0.0.1' -u$user -p$passwd --databases dbname1 dbname2 |gzip > $backpath/$serverName.sql.gz

```
#### 修改文件权限 
> chmod 744 mysqldump.sh

---
#### 创建rmbackup.sh执行文件
> vim 创建rmbackup.sh

#### 编辑脚本
```
# 删除三天前的日志
day=`date +%Y%m%d -d  -3days`
backpath=/home/mysql/timingBackup/$day

rm -rf $backpath
```

#### 修改文件权限 
> chmod 744 rmbackup.sh

---
#### 安装crontab 
> apt-get update && apt-get install cron

#### 设置定时任务
> crontab -e

```
分 时日月年 
30 0 * * * /home/mysql/timingBackup/mysqldump.sh >> /home/mysql/timingBackup/executelog.log 2>&1
35 0 * * * /home/mysql/timingBackup/rmbackup.sh >> /home/mysql/timingBackup/executelog.log 2>&1
```
#### 启动定时服务
>service cron start