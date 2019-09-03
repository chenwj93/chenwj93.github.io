---
layout: post
title: Go Proto And Truss
subtitle: go 使用proto and truss
date: 2019-09-07
categories: go
cover: 
tags: all proto go-kit 框架 
---

> 今天在自己电脑安装truss，发现还有点麻烦，故记录一下

首先truss基于proto，故先安装proto

#### 1 安装protoc
在[github](https://github.com/google/protobuf/releases)下载对应文件

解压后将protoc文件加入PATH

> $ vi /etc/profile

加入以下
```go
export PATH=$PATH:'protoc path'
```

#### 2 安装protoc-gen-go
```go
go get github.com/golang/protobuf/protoc-gen-go
cd $GOPATH/src/github.com/golang/protobuf/protoc-gen-go
go install
```
#### 3 安装protobuf库
> go get github.com/golang/protobuf/proto

#### 4 安装truss
```go
go get -u -d github.com/metaverse/truss
cd $GOPATH/src/github.com/metaverse/truss
make dependencies
make
```

安装时有些地址被墙，导致下载失败

那么我们可以增加goproxy环境变量，参考[https://goproxy.io/](https://goproxy.io/)

增加环境变量
```go
# Enable the go modules feature
export GO111MODULE=on
# Set the GOPROXY environment variable
export GOPROXY=https://goproxy.io
```

安装完成后，执行生成代码命令，报错，缺少go-bindata

github查找go-bindata项目，clone代码，然后执行go install，安装完成
