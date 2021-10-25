# gRPC: Implement rate limit on server side

## Introduction
Implement rate limit interceptor/middleware in gRPC micro-service easily with [rk-boot](https://github.com/rookie-ninja/rk-boot).

## Introduce rk-boot
We introduce [rk-boot](https://github.com/rookie-ninja/rk-boot) which is a library can be used to create golang microservice with grpc in a convenient way.
- [Docs](https://rkdev.info/docs/bootstrapper/getting-started/grpc-golang/)
- [Source code](https://github.com/rookie-ninja/rk-boot)
- [Example](https://github.com/rookie-ninja/rk-demo/tree/master/grpc/getting-started)

## Install rk-boot
```go
go get github.com/rookie-ninja/rk-boot
```

## Quick start
Please visit rkdev.info for detailed document.

### I. Create boot.yaml
We enabled commonService which contains a couple commonly used API like /rk/v1/healthy.

Let's limit rate on /rk.api.v1.RkCommonService/Healthy to be 0 and the other to 100.

**By default, rk-boot will enable grpc-gateway automatically.**

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

### II. Create main.go 
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

### III. Directory hierarchy
```
$ tree
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

### IV. Start main.go
```
$ go run main.go
```

### V. Validate
- Send request to /rk/v1/healthy which will be throttled.

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

- Send request to /rk/v1/info which will be normal.

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

## YAML options

| 名字 | 描述 | 类型 | 默认值 |
| ------ | ------ | ------ | ------ |
| grpc.interceptors.rateLimit.enabled | 启动限流兰姐去 | boolean | false |
| grpc.interceptors.rateLimit.algorithm | 限流算法， 支持 tokenBucket 和 leakyBucket | string | tokenBucket |
| grpc.interceptors.rateLimit.reqPerSec | 全局限流值 | int | 0 |
| grpc.interceptors.rateLimit.paths.path | gRPC 方法路径 | string | "" |
| grpc.interceptors.rateLimit.paths.reqPerSec | 基于 gRPC 方法路径的限流值 | int | 0 |

