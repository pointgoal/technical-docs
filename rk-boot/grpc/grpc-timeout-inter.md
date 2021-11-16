# gRPC: 实现 gRPC 超时中间件

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 快速搭建 gRPC 超时中间件。

> **什么是 gRPC 超时中间件？** 
>
> 中间件会拦截 gRPC 请求，并根据策略返回超时错误。

**请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 启动的 gRPC 服务。

支持全局超时和 API 超时设定。

### 1.创建 boot.yaml
boot.yaml 文件告诉 rk-boot 如何启动 gRPC 服务。

为了验证，我们启动了 commonService，commonService 里包含了一系列常用 API，例如 /rk/v1/gc。

设定全局超时为 5秒，让 GC 的超时时间定位 1 毫秒，GC 一般会超过 1 毫秒。

```yaml
---
grpc:
  - name: greeter                                   # Required
    port: 8080                                      # Required
    enabled: true                                   # Required
    commonService:
      enabled: true                                 # Optional, Enable common service for testing
    interceptors:
      timeout:
        enabled: true                               # Optional, default: false
        timeoutMs: 5000                             # Optional, default: 5000
        paths: 
          - path: "/rk.api.v1.RkCommonService/Gc"   # Optional, default: ""
            timeoutMs: 1                            # Optional, default: 5000
```

c

### 3.启动 main.go
```
$ go run main.go
```

### 4.验证
发送 GC 请求。

```
$ grpcurl -plaintext localhost:8080 rk.api.v1.RkCommonService.Gc
ERROR:
  Code: Canceled
  Message: Request timed out!
  Details:
  1)	{"@type":"type.googleapis.com/rk.api.v1.ErrorDetail","code":1,"message":"[from-grpc] Request timed out!","status":"Canceled"}
```

```
$ curl -X GET localhost:8080/rk/v1/gc
{
    "error":{
        "code":408,
        "status":"Request Timeout",
        "message":"Request timed out!",
        "details":[
            {
                "code":1,
                "status":"Canceled",
                "message":"[from-grpc] Request timed out!"
            }
        ]
    }
}
```