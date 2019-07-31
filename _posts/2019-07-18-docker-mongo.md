---
layout: post
title: Docker deploy mongo
subtitle: docker 部署mongodb
date: 2019-07-25
categories: docker
cover: 
tags: all docker mongo mongodb
---

> 库存 docker-mongo

### 部署
##### 拉取mongo镜像
>docker pull mongo

##### 创建容器
```
docker run  \
--name mongodb \
-p 27017:27017  \
-v /mongo/configdb:/data/configdb/ \
-v /mongo/db/:/data/db/ \
-d mongo --auth
```

### 创建管理员账号
> docker exec -it mongodb mongo admin

> db.createUser({ user: 'usertest', pwd: 'pwdtest', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });

**管理员账号并非超级账号，管理员只能管理用户，不能直接操作其它用户的数据库**
### 创建普通账号
> docker exec -it mongodb mongo admin

##### 认证
> db.auth("usertest","pwdtest");

##### 创建db school
> use school;

##### 创建school的读写账号
> db.createUser({ user: 'test', pwd: 'test', roles: [ { role: "readWrite", db: "school" } ] });

### 删除用户
> db.dropUser("用户名")