---
layout: post
title: Docker Deploy Gitbook
subtitle: docker 部署gitbook服务
date: 2019-08-15
categories: docker
cover: 
tags: 
---

> 记录一个小知识

gitbook就不介绍了，这几天看GitHub上的一本开源书，在线看有时候会比较慢，部署到本地会好一些

由于我已经安装了docker，所以采用docker部署比较方便

#### 创建文件夹
```
mkdir -p /Data/bookName/gitbook
cd /Data/bookName
mkdir html
```
#### pull镜像
> docker pull fellah/gitbook

#### 部署
```
docker run --name gitbook \
-p 4000:4000  \
-v /Data/bookName/gitbook:/srv/gitbook \
-v /Data/bookName/html:/srv/html \
fellah/gitbook
```