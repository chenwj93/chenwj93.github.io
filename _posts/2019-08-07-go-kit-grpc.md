---
layout: post
title: Go Kit GRPC
subtitle: go-kit gRpc
date: 2019-08-07
categories: go
cover: 
tags: all rpc go-kit 框架 微服务
---

> 读《Go语言高级编程》笔记

gRPC服务一般用于集群内部通信，如果需要对外暴露服务一般会提供等价的REST接口。

go-kit开发工具包对此支持比较好,同时搭配truss代码生成工具，可以帮助我们解决繁琐的搭建流程，只需要专注于业务实现

例如以下`test.proto`:
```go
syntax = "proto3";

package test;

import "github.com/metaverse/truss/deftree/googlethirdparty/annotations.proto";

service Test {
    rpc HealthCheck (HealthCheckRequest) returns (HealthCheckResponse) {
        option (google.api.http) = {
            get: "/health"
        };
    }

    rpc Code2Session (Code2SessionRequest) returns (Code2SessionResponse) {
        option (google.api.http) = {
            post: "/test/v1/code2Session"
        };
	}
}

message HealthCheckRequest {
}

message HealthCheckResponse {
    bool status = 1;
}

message Code2SessionRequest{
    string code = 1;
}

message Code2SessionResponse{
        int64 userId = 1;
        string miniOpenId = 2;
}
```
执行命令：
> truss test.proto

将生成以下代码结构

<img src="/img/gokittruss.png">

- cmd 目录包含了启动main函数
- handlers 目录包含了业务代码，也是我们可以编辑的目录（其它目录在重新生成代码时会被覆盖）
- svc 目录包含了基础服务，定义了服务接口

服务启动代码示例：
```go 
func Run(cfg Config) {
	endpoints := NewEndpoints()

	// Mechanical domain.
	errc := make(chan error)

	// Interrupt handler.
	go handlers.InterruptHandler(errc)

	// Debug listener.
	go func() {
		log.Println("transport", "debug", "addr", cfg.DebugAddr)

		m := http.NewServeMux()
		m.Handle("/debug/pprof/", http.HandlerFunc(pprof.Index))
		m.Handle("/debug/pprof/cmdline", http.HandlerFunc(pprof.Cmdline))
		m.Handle("/debug/pprof/profile", http.HandlerFunc(pprof.Profile))
		m.Handle("/debug/pprof/symbol", http.HandlerFunc(pprof.Symbol))
		m.Handle("/debug/pprof/trace", http.HandlerFunc(pprof.Trace))

		errc <- http.ListenAndServe(cfg.DebugAddr, m)
	}()

	// HTTP transport.
	go func() {
		log.Println("transport", "HTTP", "addr", cfg.HTTPAddr)
		h := svc.MakeHTTPHandler(endpoints)
		errc <- http.ListenAndServe(cfg.HTTPAddr, h)
	}()

	// gRPC transport.
	go func() {
		log.Println("transport", "gRPC", "addr", cfg.GRPCAddr)
		ln, err := net.Listen("tcp", cfg.GRPCAddr)
		if err != nil {
			errc <- err
			return
		}

		srv := svc.MakeGRPCServer(endpoints)
		s := grpc.NewServer()
		pb.RegisterTestServer(s, srv)

		errc <- s.Serve(ln)
	}()

	// Run!
	log.Println("exit", <-errc)
}
```