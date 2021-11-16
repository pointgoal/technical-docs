# Echo 框架：实现服务端限流中间件

## 介绍
通过一个完整例子，在基于 Echo 框架的微服务中实现【限流】中间件。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Echo 框架的微服务。

请访问如下地址获取完整教程：https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
### 1.创建 boot.yaml
为了验证，我们启动了如下几个选项：
- **commonService**：commonService 里包含了一系列通用 API。[详情](https://github.com/rookie-ninja/rk-echo#common-service-1)

我们针对 /rk/v1/healthy 进行限流处理，设置成 0，其他 API 设置成 100。

```yaml
---
echo:
  - name: greeter                     # Required
    port: 8080                        # Required
    enabled: true                     # Required
    commonService:
      enabled: true                   # Optional, default: false
    interceptors:
      rateLimit:
        enabled: true
        reqPerSec: 100
        paths:
          - path: "/rk/v1/healthy"
            reqPerSec: 0
```

### 2.创建 main.go 
```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
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
$ curl -X GET localhost:8080/rk/v1/healthy
{
    "error":{
        "code":429,
        "status":"Too Many Requests",
        "message":"",
        "details":[
            "slow down your request"
        ]
    }
}
```

- 发送请求到 /rk/v1/info，正常

```
$ curl -X GET localhost:8080/rk/v1/info
{
    "appName":"rk-demo",
    "version":"master-2c9c6fd",
    "description":"Internal RK entry which describes application with fields of appName, version and etc.",
    "keywords":[],
    "homeUrl":"",
    "iconUrl":"",
    "docsUrl":[],
    "maintainers":[],
    "uid":"501",
    "gid":"20",
    "username":"Dongxun Yin",
    "startTime":"2021-11-14T00:32:09+08:00",
    "upTimeSec":123,
    "upTimeStr":"2 minutes",
    "region":"",
    "az":"",
    "realm":"",
    "domain":""
}
```

## YAML 选项

| 名字 | 描述 | 类型 | 默认值 |
| ------ | ------ | ------ | ------ |
| echo.interceptors.rateLimit.enabled | 启动限流中间件 | boolean | false |
| echo.interceptors.rateLimit.algorithm | 限流算法， 支持 tokenBucket 和 leakyBucket | string | tokenBucket |
| echo.interceptors.rateLimit.reqPerSec | 全局限流值 | int | 0 |
| echo.interceptors.rateLimit.paths.path | HTTP 路径 | string | "" |
| echo.interceptors.rateLimit.paths.reqPerSec | 基于 HTTP 路径的限流值 | int | 0 |

