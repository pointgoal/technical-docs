# Gin 框架: 自动添加 RequestId

## 介绍
通过一个完整例子，在基于 Gin 框架中，为每一个 API 自动添加 RequestId 。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Gin 框架微服务。

请访问如下地址获取完整教程：

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
开启了 meta 中间件之后，每一个请求都会自动包含如下值。

| Header 键 | 详情 |
| ---- | ---- |
| X-Request-Id | 中间件会自动生成请求 ID。|
| X-[Prefix]-App | 服务名称。 |
| X-[Prefix]-App-Version | 服务版本。 |
| X-[Prefix]-App-Unix-Time | 当前服务的 Unix 时间。 |
| X-[Prefix]-Request-Received-Time | 接收到请求的时间戳。 |

### 1.创建 boot.yaml
为了验证，我们启动了 commonService，commonService 里包含了一系列常用 API，例如 /rk/v1/healthy。

```
---
gin:
  - name: greeter                   # Required
    port: 8080                      # Required
    enabled: true                   # Required
    commonService:
      enabled: true                 # Optional, enable common service
    interceptors:
      meta:
        enabled: true               # Optional, enable meta middleware
```

### 2.创建 main.go
```
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

```
$ curl -vs -X GET localhost:8080/rk/v1/healthy
  ...
  < HTTP/1.1 200 OK
  < Content-Type: application/json; charset=utf-8
  < X-Request-Id: 78c25a06-3b34-4ecb-b9dd-7197078873c7
  < X-Rk-App-Name: rk-demo
  < X-Rk-App-Unix-Time: 2021-11-21T00:24:49.662023+08:00
  < X-Rk-App-Version: master-2c9c6fd
  < X-Rk-Received-Time: 2021-11-21T00:24:49.662023+08:00
  ...
  {"healthy":true}
```

## 覆盖 requestId
如果我们希望自定义 requestId，需要添加一个 Header。

```
func Greeter(ctx *gin.Context) {
	// Override request id
	rkginctx.SetHeaderToClient(ctx, rkginctx.RequestIdKey, "request-id-override")
	// We expect new request id attached to logger
	rkginctx.GetLogger(ctx).Info("Received request")

	ctx.JSON(http.StatusOK, &GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", ctx.Query("name")),
	})
}

// Response.
type GreeterResponse struct {
	Message string
}
```

RequestId 会被覆盖。

```
$ curl -vs -X GET "localhost:8080/v1/greeter?name=rk-dev"
...
< X-Request-Id: request-id-override
< X-Rk-App-Name: rk-demo
< X-Rk-App-Unix-Time: 2021-11-21T00:37:16.514685+08:00
< X-Rk-App-Version: master-2c9c6fd
< X-Rk-Received-Time: 2021-11-21T00:37:16.514685+08:00
...
{"Message":"Hello rk-dev!"}
```

> 如果我们启动了日志中间件，那我们会看到如下的日志。

```
2021-11-21T00:39:04.605+0800    INFO    basic/main.go:54        Received request        {"requestId": "request-id-override"}
```

```
------------------------------------------------------------------------
endTime=2021-11-21T00:39:04.605647+08:00
startTime=2021-11-21T00:39:04.605483+08:00
elapsedNano=164096
timezone=CST
ids={"eventId":"request-id-override","requestId":"request-id-override"}
app={"appName":"rk-demo","appVersion":"master-2c9c6fd","entryName":"greeter","entryType":"GinEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"192.168.101.5","os":"darwin","realm":"*","region":"*"}
payloads={"apiMethod":"GET","apiPath":"/v1/greeter","apiProtocol":"HTTP/1.1","apiQuery":"name=rk-dev","userAgent":"curl/7.64.1"}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost:61967
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```

## 覆盖 header 前缀

```
---
gin:
  - name: greeter
    ...
    interceptors:
      meta:
        enabled: true        # Enable meta middleware
        prefix: "Override"   # Override prefix which formed as X-[Prefix]-xxx
```

```
$ curl -vs -X GET localhost:8080/rk/v1/healthy
...
< X-Override-App-Name: rk-demo
< X-Override-App-Unix-Time: 2021-11-21T00:40:15.761993+08:00
< X-Override-App-Version: master-2c9c6fd
< X-Override-Received-Time: 2021-11-21T00:40:15.761993+08:00
< X-Request-Id: 1449deb5-464d-4b65-8430-985413c2671b
...
{"healthy":true}
```
