# GRPC: 实现服务端限流

## 介绍

本文将介绍如何在 gRPC 微服务中实现【限流】拦截器/中间件。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 gRPC 服务。

> **请访问如下地址获取完整教程：**
>
> - https://rkdev.info/cn
>
> - https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
### 1.创建 boot.yaml
为了验证，我们启动了 commonService，commonService 里包含了一系列常用 API，例如 /rk/v1/healthy。

我们针对 /rk.api.v1.RkCommonService/Healthy 进行限流处理，设置成 0，其他 API 设置成 100。

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    commonService:
      enabled: true                 # Enable common service for testing
    interceptors:
      rateLimit:
        enabled: true
        reqPerSec: 100
#        algorithm: "leakyBucket"
        paths:
          - path: "/rk.api.v1.RkCommonService/Healthy"
            reqPerSec: 0
```

### 2.创建 main.go 
```
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.文件夹结构 
```
$ tree
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

### 4.启动 main.go
```
$ go run main.go
```

### 5.验证
- 发送请求到 /rk/v1/healthy，会被限流。

```
{
    "code":8,
    "message":"Slow down your request.",
    "details":[
        {
            "@type":"type.googleapis.com/rk.api.v1.ErrorDetail",
            "code":8,
            "status":"ResourceExhausted",
            "message":"[from-grpc] Slow down your request."
        }
    ]
}
```

- 发送请求到 /rk/v1/info，正常

```
{
    "appName":"demo",
    "az":"",
    "description":"Internal RK entry which describes application with fields of appName, version and etc.",
    "docsUrl":[

    ],
    "domain":"",
    "gid":"20",
    "homeUrl":"",
    "iconUrl":"",
    "keywords":[

    ],
    "maintainers":[

    ],
    "realm":"",
    "region":"",
    "startTime":"2021-10-25T01:15:48+08:00",
    "uid":"501",
    "upTimeSec":76,
    "upTimeStr":"1 minute",
    "username":"Dongxun Yin",
    "version":"master-557da30"
}
```

## YAML 选项

| 名字 | 描述 | 类型 | 默认值 |
| ------ | ------ | ------ | ------ |
| grpc.interceptors.rateLimit.enabled | 启动限流兰姐去 | boolean | false |
| grpc.interceptors.rateLimit.algorithm | 限流算法， 支持 tokenBucket 和 leakyBucket | string | tokenBucket |
| grpc.interceptors.rateLimit.reqPerSec | 全局限流值 | int | 0 |
| grpc.interceptors.rateLimit.paths.path | gRPC 方法路径 | string | "" |
| grpc.interceptors.rateLimit.paths.reqPerSec | 基于 gRPC 方法路径的限流值 | int | 0 |





