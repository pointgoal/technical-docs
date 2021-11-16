# Gin 框架：实现超时中间件

## 介绍
通过一个完整例子，在基于 Gin 框架的微服务中实现【超时】中间件。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Gin 框架的微服务。

请访问如下地址获取完整教程：

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
支持全局超时和 API 超时设定。

### 1.创建 boot.yaml
boot.yaml 文件告诉 rk-boot 如何启动 Gin 服务。

为了验证，我们启动了如下几个选项：
- **commonService**：commonService 里包含了一系列通用 API。[详情](https://github.com/rookie-ninja/rk-gin#common-service-1)

设定全局超时为 5秒，让 GC 的超时时间为 1 毫秒，GC 一般会超过 1 毫秒。

```yaml
---
gin:
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
          - path: "/rk/v1/gc"                       # Optional, default: ""
            timeoutMs: 1                            # Optional, default: 5000
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

### 3.启动 main.go
```
$ go run main.go
```

### 4.验证
发送 GC 请求。

```
$ curl -X GET localhost:8080/rk/v1/gc
{
    "error":{
        "code":408,
        "status":"Request Timeout",
        "message":"Request timed out!",
        "details":[]
    }
}
```
