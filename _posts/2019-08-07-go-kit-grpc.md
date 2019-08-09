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

## 重要知识点
### OPTIONS请求
相信每一个web开发者都碰到过options跨域问题
#### 第一种
第一种方法可以通过nginx进行过滤
```go
if ($request_method = 'OPTIONS') {
   add_header 'Access-Control-Allow-Origin' '*';
   add_header 'Access-Control-Allow-Credentials' 'true';
   add_header 'Access-Control-Allow-Methods' 'GET, POST, PATCH, DELETE, PUT, OPTIONS';
   add_header 'Access-Control-Allow-Headers' 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-ContContent-Type,  Access-Control-Expose-Headers, Token, Authorization';
   add_header 'Access-Control-Max-Age' 1728000;
   add_header 'Content-Type' 'text/plain charset=UTF-8';
   add_header 'Content-Length' 0;
   return 204;
}
```

#### 第二种
另一种就是在代码路由中进行处理

然而`proto`+`go-kit`+`truss` 这一套开发工具中，serverOptions 只能做数据处理，无法干涉执行流程。而middleware 则在路由解析流程之后。

那么是否可以定义一条路由规则（OPTIONS的所有URI）呢，我在protobuf中没有找到解决方法。

那只有最后一条路了，修改生成的代码

首先，在util包中增加一个函数
```go
func OptionsFileter(m *mux.Router) {
	m.Methods("OPTIONS").PathPrefix("/").HandlerFunc(
		func(writer http.ResponseWriter, request *http.Request) {
			writer.WriteHeader(204)
			writer.Header().Set("Access-Control-Allow-Origin", "*")               //允许访问所有域
			writer.Header().Add("Access-Control-Allow-Headers", "*")   //header的类型
			writer.Header().Set("content-type", "application/json;charset=utf-8") //返回数据格式是json
		})
}
```
然后修改`transport_http.go`
```go
func MakeHTTPHandler(endpoints Endpoints) http.Handler {
	serverOptions := []httptransport.ServerOption{
		httptransport.ServerBefore(headersToContext),
		httptransport.ServerErrorEncoder(errcode.HttpEncoder),
		httptransport.ServerAfter(httptransport.SetContentType(contentType)),
	}
	m := mux.NewRouter()
    http_util.OptionsFileter(m) // 新增这一个路由

	m.Methods("GET").Path("/health").Handler(httptransport.NewServer(
		endpoints.HealthCheckEndpoint,
		DecodeHTTPHealthCheckZeroRequest,
		EncodeHTTPGenericResponse,
		serverOptions...,
	))

	...
}
```

由于我们是使用shell脚本调truss生成代码，同时会适当做出一些代码修改，所以可以将这个修改操作加入到脚本中
```go
    sed -i '/m := mux.NewRouter()/a\    http_util.OptionsFileter(m)' svc/transport_http.go 
    sed -i '/^import (/a\    http_util "code.raying.com/cloud/utils/gokit-interceptor/http"' svc/transport_http.go
```

#### 第三种
看gorilla/mux 突然发现mux本身也自带一个middleware，那么就有了这第三种解决方法

首先在util包中增加mux_filter.go：
```go
func OptionsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Do stuff here
		if r.Method == http.MethodOptions {
			utils.ULog.Println(http.MethodOptions, r.URL)
			w.WriteHeader(204)
			w.Header().Set("Access-Control-Allow-Origin", "*")               //允许访问所有域
			w.Header().Add("Access-Control-Allow-Headers", "*")   //header的类型
			w.Header().Set("content-type", "application/json;charset=utf-8") //返回数据格式是json
			return
		}

		// Call the next handler, which can be another middleware in the chain, or the final handler.
		next.ServeHTTP(w, r)
	})
}

```

然后修改启动方法
```go
func Run(cfg Config) {
	endpoints := NewEndpoints()

	...

	// HTTP transport.
	go func() {
		log.Println("transport", "HTTP", "addr", cfg.HTTPAddr)
		h := svc.MakeHTTPHandler(endpoints)
		m := mux.NewRouter()
		m.PathPrefix("/").Handler(h)
		m.Use(http_util.OptionsMiddleware)
		errc <- http.ListenAndServe(cfg.HTTPAddr, m)
	}()

	...
}
```