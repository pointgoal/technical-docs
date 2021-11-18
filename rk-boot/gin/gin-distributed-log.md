# Gin 框架：实现分布式日志追踪

## 介绍
通过一个完整例子，在基于 Gin 框架中实现分布式日志追踪。

> **什么是 API 日志追踪？**
>
> 一个 API 请求会跨多个微服务，我们希望通过一个唯一的 ID 检索到整个链路的日志。

![image](img/trace-arch.png)

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Gin 框架的微服务。

请访问如下地址获取完整教程：

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
我们会创建 /v1/greeter API 进行验证，同时开启 logging, meta 和 tracing 中间件以达到目的。

### 1. 创建 bootA.yaml & serverA.go
ServerA 监听 1949 端口，并且发送请求给 ServerB。

我们通过 rkginctx.InjectSpanToNewContext() 方法把 Tracing 信息注入到 Context 中，发送给 ServerB。 

```yaml
---
gin:
  - name: greeter                   # Required
    port: 1949                      # Required
    enabled: true                   # Required
    interceptors:
      loggingZap:
        enabled: true               # Optional, enable logging interceptor
      meta:
        enabled: true               # Optional, enable meta interceptor
      tracingTelemetry:
        enabled: true               # Optional, enable tracing interceptor
```

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/gin-gonic/gin"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-gin/interceptor/context"
	"net/http"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot(rkboot.WithBootConfigPath("bootA.yaml"))

	// Register handler
	boot.GetGinEntry("greeter").Router.GET("/v1/greeter", GreeterA)

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

// GreeterA will add trace info into context and call serverB
func GreeterA(ctx *gin.Context) {
	// Call serverB at 2008
	req, _ := http.NewRequest(http.MethodGet, "http://localhost:2008/v1/greeter", nil)

	// Inject current trace information into context
	rkginctx.InjectSpanToHttpRequest(ctx, req)

	// Call server
	http.DefaultClient.Do(req)
	
	// Respond to request
	ctx.String(http.StatusOK, "Hello from serverA!")
}
```

### 2. 创建 bootB.yaml & serverB.go
ServerB 监听 2008 端口。

```yaml
---
gin:
  - name: greeter                   # Required
    port: 2008                      # Required
    enabled: true                   # Required
    interceptors:
      loggingZap:
        enabled: true               # Optional, enable logging interceptor
      meta:
        enabled: true               # Optional, enable meta interceptor
      tracingTelemetry:
        enabled: true               # Optional, enable tracing interceptor
```

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/gin-gonic/gin"
	"github.com/rookie-ninja/rk-boot"
	"net/http"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot(rkboot.WithBootConfigPath("bootB.yaml"))

	// Register handler
	boot.GetGinEntry("greeter").Router.GET("/v1/greeter", GreeterB)

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

// GreeterB receive request from serverA and respond.
func GreeterB(ctx *gin.Context) {
	ctx.String(http.StatusOK, "Hello from serverB!")
}
```

### 3. 文件夹结构

```
.
├── bootA.yaml
├── bootB.yaml
├── go.mod
├── go.sum
├── serverA.go
└── serverB.go

0 directories, 6 files
```

### 4. 启动 ServerA & ServerB

```
$ go run serverA.go
$ go run serverB.go
```

### 5. 往 ServerA 发送请求

```
$ curl localhost:1949/v1/greeter
Hello from serverA!
```

### 6. 验证日志

两个服务的日志中，会有同样的 traceId，不同的 requestId。

我们可以通过 grep traceId 来追踪 RPC。

- ServerA

```
------------------------------------------------------------------------
endTime=2021-11-18T01:29:56.698997+08:00
...
ids={"eventId":"f5878390-1a5a-4bb9-8b39-bf4261864c0f","requestId":"f5878390-1a5a-4bb9-8b39-bf4261864c0f","traceId":"b2d70ab9f8207ef4a9f0c3fb1be5c22c"}
...
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```

- ServerB

```
------------------------------------------------------------------------
endTime=2021-11-18T01:29:56.698606+08:00
...
ids={"eventId":"273c97d2-e11a-46f5-a044-bb9c0cf64540","requestId":"273c97d2-e11a-46f5-a044-bb9c0cf64540","traceId":"b2d70ab9f8207ef4a9f0c3fb1be5c22c"}
...
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```

## 概念
当我们没有使用例如 jaeger 调用链服务的时候，我们希望通过日志来追踪分布式系统里的 RPC 请求。

rk-boot 的中间件会通过 openTelemetry 库来向日志写入 traceId 来追踪 RPC。

当启动了日志中间件，原数据中间件，调用链中间件的时候，中间件会往日志里写入如下三种 ID。

### EventId
当启动了日志中间件，EventId 会自动生成。

```yaml
---
gin:
  - name: greeter                   # Required
    port: 1949                      # Required
    enabled: true                   # Required
    interceptors:
      loggingZap:
        enabled: true
```

```
------------------------------------------------------------------------
...
ids={"eventId":"cd617f0c-2d93-45e1-bef0-95c89972530d"}
...
```

### RequestId
当启动了日志中间件和原数据中间件，RequestId 和 EventId 会自动生成，并且这两个 ID 会一致。

```yaml
---
gin:
  - name: greeter                   # Required
    port: 1949                      # Required
    enabled: true                   # Required
    interceptors:
      loggingZap:
        enabled: true
      meta:
        enabled: true
```

```
------------------------------------------------------------------------
...
ids={"eventId":"8226ba9b-424e-4e19-ba63-d37ca69028b3","requestId":"8226ba9b-424e-4e19-ba63-d37ca69028b3"}
...
```

> 即使用户覆盖了 RequestId，EventId 也会保持一致。

```
rkginctx.AddHeaderToClient(ctx, rkginctx.RequestIdKey, "overridden-request-id")
```

```
------------------------------------------------------------------------
...
ids={"eventId":"overridden-request-id","requestId":"overridden-request-id"}
...
```

### TraceId
当启动了调用链中间件，traceId 会自动生成。

```yaml
---
gin:
  - name: greeter                   # Required
    port: 1949                      # Required
    enabled: true                   # Required
    interceptors:
      loggingZap:
        enabled: true
      meta:
        enabled: true
      tracingTelemetry:
        enabled: true
```

```
------------------------------------------------------------------------
...
ids={"eventId":"dd19cf9a-c7be-486c-b29d-7af777a78ebe","requestId":"dd19cf9a-c7be-486c-b29d-7af777a78ebe","traceId":"316a7b475ff500a76bfcd6147036951c"}
...
```
