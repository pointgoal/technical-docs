# GoFrame 框架：添加 API 日志中间件

## 介绍
通过一个完整例子，在基于 GoFrame 框架的微服务中添加 API 日志中间件。

> 什么是日志拦截器/中间件？
>
> 日志拦截器会对每一个 API 请求记录日志。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 GoFrame 微服务。

请访问如下地址获取完整教程：https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot/gf
```

## 快速开始
[rk-boot](https://github.com/rookie-ninja/rk-boot) 默认集成如下两个开源库。
 
- [uber-go/zap](https://github.com/uber-go/zap) 作为底层的日志库。
- [logrus](https://github.com/sirupsen/logrus) 作为日志滚动。

### 1.创建 boot.yaml
boot.yaml 文件描述了 GoFrame 框架启动的原信息，rk-boot 通过读取 boot.yaml 来启动 GoFrame。

为了验证，我们同时启动了 commonService。commonService 里包含了一系列通用 API。
详情: [CommonService](https://github.com/rookie-ninja/rk-gf#common-service-1)

```yaml
---
gf:
  - name: greeter                   # Required, name of GoFrame entry
    port: 8080                      # Required, port of GoFrame entry
    enabled: true                   # Required, enable GoFrame entry
    commonService:
      enabled: true                 # Optional, enable common service
    interceptors:
      loggingZap:
        enabled: true               # Optional, enable logging interceptor
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
	"fmt"
	"github.com/gogf/gf/v2/net/ghttp"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-boot/gf"
	"net/http"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Register handler
	gfEntry := rkbootgf.GetGfEntry("greeter")
	gfEntry.Server.BindHandler("/v1/greeter", func(ctx *ghttp.Request) {
		ctx.Response.WriteHeader(http.StatusOK)
		ctx.Response.WriteJson(&GreeterResponse{
			Message: fmt.Sprintf("Hello %s!", ctx.GetQuery("name")),
		})
	})

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

type GreeterResponse struct {
	Message string
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

2021-12-29T14:26:21.361+0800    INFO    boot/gf_entry.go:1050   Bootstrap gfEntry       {"eventId": "863301c4-f75f-49db-a421-8f992f4a5001", "entryName": "greeter"}
------------------------------------------------------------------------
endTime=2021-12-29T14:26:21.361609+08:00
startTime=2021-12-29T14:26:21.361279+08:00
elapsedNano=329215
timezone=CST
ids={"eventId":"863301c4-f75f-49db-a421-8f992f4a5001"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"commonServiceEnabled":true,"commonServicePathPrefix":"/rk/v1/","gfPort":8080}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost
operation=Bootstrap
resCode=OK
eventStatus=Ended
EOE
```

### 5.验证
我们发送 CommonService 自带的 /rk/v1/healthy 请求。

```
$ curl -X GET localhost:8080/rk/v1/healthy
{"healthy":true}
```

发送请求到 /v1/greeter。

```
$ curl "localhost:8080/v1/greeter?name=rk-dev"
{"Message":"Hello rk-dev!"}
```

EventLog 会默认输出到 stdout。

下面的日志格式来自 [rk-query](https://github.com/rookie-ninja/rk-query) ，用户也可以选择 JSON 格式，我们稍后会介绍。

```
------------------------------------------------------------------------
endTime=2021-12-29T14:27:07.263385+08:00
startTime=2021-12-29T14:27:07.263225+08:00
elapsedNano=159773
timezone=CST
ids={"eventId":"a763fe44-a95d-4151-9db0-c90aaab7f030"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"apiMethod":"GET","apiPath":"/v1/greeter","apiProtocol":"HTTP/1.1","apiQuery":"name=rk-dev","userAgent":"curl/7.64.1"}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost:54360
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```

## 修改日志格式
我们可以通过修改 boot.yaml 来修改日志格式。
目前支持 json 和 console 两种格式，默认为 console。

通过修改 eventLoggerEncoding 的值为 json，我们可以把日志的输出为 JSON 格式。

```yaml
gf:
  - name: greeter                    # Required, name of GoFrame entry
    port: 8080                       # Required, port of GoFrame entry
    enabled: true                    # Required, enable GoFrame entry
    commonService:
      enabled: true                  # Optional, enable common service
    interceptors:
      loggingZap:
        enabled: true                # Optional, enable logging interceptor
        zapLoggerEncoding: "json"    # Override to json format, option: json or console
        eventLoggerEncoding: "json"  # Override to json format, option: json or console
```

```json
{
    "endTime":"2021-12-29T14:29:07.204+0800",
    "startTime":"2021-12-29T14:29:07.204+0800",
    "elapsedNano":188838,
    "timezone":"CST",
    "ids":{
        "eventId":"5d7f4889-534e-4ed7-98b7-d9fb8a9f20e9"
    },
    "app":{
        "appName":"rk",
        "appVersion":"",
        "entryName":"greeter",
        "entryType":"GfEntry"
    },
    "env":{
        "arch":"amd64",
        "az":"*",
        "domain":"*",
        "hostname":"lark.local",
        "localIP":"10.8.0.2",
        "os":"darwin",
        "realm":"*",
        "region":"*"
    },
    "payloads":{
        "apiMethod":"GET",
        "apiPath":"/v1/greeter",
        "apiProtocol":"HTTP/1.1",
        "apiQuery":"name=rk-dev",
        "userAgent":"curl/7.64.1"
    },
    "error":{},
    "counters":{},
    "pairs":{},
    "timing":{},
    "remoteAddr":"localhost:62259",
    "operation":"/v1/greeter",
    "eventStatus":"Ended",
    "resCode":"200"
}
```

## 修改日志路径
通过修改 eventLoggerOutputPaths 的值，可以指定输出路径。

日志默认在 1GB 之后，进行切割，并压缩。

```yaml
---
gf:
  - name: greeter                                     # Required, name of GoFrame entry
    port: 8080                                        # Required, port of GoFrame entry
    enabled: true                                     # Required, enable GoFrame entry
    commonService:
      enabled: true                                   # Optional, enable common service
    interceptors:
      loggingZap:
        enabled: true                                 # Optional, enable logging interceptor
        zapLoggerOutputPaths: ["logs/app.log"]        # Override output paths
        eventLoggerOutputPaths: ["logs/event.log"]    # Override output paths
```

```
.
├── boot.yaml
├── go.mod
├── go.sum
├── logs
│   └── event.log
└── main.go
```

## 概念
验证了日志拦截器，我们来具体讲一下 rk-boot 提供的日志拦截器都有哪些功能。

我们需要提前了解两个概念。
- EventLogger
- ZapLogger

### ZapLogger
用于记录错误/详细日志，用户可以获取本次 RPC 调用的 ZapLogger 实例，进行日志写入，每个 RPC 的 ZapLogger 实例都包含当前的 RequestId。

```
2021-12-29T14:30:43.655+0800    INFO    boot/gf_entry.go:1050   Bootstrap gfEntry       {"eventId": "79a17d94-9d0d-44f8-abf7-84f9dfb055e0", "entryName": "greeter"}
```

### EventLogger
RK 启动器把每一个 RPC 请求视作 **Event**，并且使用 rk-query 中的 Event 类型来记录日志。

| 字段 | 详情 |
| ---- | ---- |
| endTime | 结束时间 |
| startTime | 开始时间 |
| elapsedNano | Event 时间开销（Nanoseconds） |
| timezone | 时区 |
| ids | 包含 eventId, requestId 和 traceId。如果原数据拦截器被启动，或者 event.SetRequest() 被用户调用，新的 RequestId 将会被使用，同时 eventId 与 requestId 会一模一样。 如果调用链拦截器被启动，traceId 将会被记录。|
| app | 包含 [appName, appVersion](https://github.com/rookie-ninja/rk-entry#appinfoentry), entryName, entryType。 |
| env | 包含 arch, az, domain, hostname, localIP, os, realm, region. realm, region, az, domain 字段。这些字段来自系统环境变量（REALM，REGION，AZ，DOMAIN）。 "*" 代表环境变量为空。|
| payloads | 包含 RPC 相关信息。 |
| error | 包含错误。|
| counters | 通过 event.SetCounter() 来操作。|
| pairs | 通过 event.AddPair() 来操作。 |
| timing | 通过 event.StartTimer() 和 event.EndTimer() 来操作。 |
| remoteAddr | RPC 远程地址。 |
| operation | RPC 名字。 |
| resCode | RPC 返回码。 |
| eventStatus | Ended 或者 InProgress |

```
------------------------------------------------------------------------
endTime=2021-12-29T14:27:07.263385+08:00
startTime=2021-12-29T14:27:07.263225+08:00
elapsedNano=159773
timezone=CST
ids={"eventId":"a763fe44-a95d-4151-9db0-c90aaab7f030"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"apiMethod":"GET","apiPath":"/v1/greeter","apiProtocol":"HTTP/1.1","apiQuery":"name=rk-dev","userAgent":"curl/7.64.1"}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost:54360
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```

## 日志中间件选项
| 名字 | 描述 | 类型 | 默认值 |
| ------ | ------ | ------ | ------ |
| gf.interceptors.loggingZap.enabled | 启动日志拦截器 | boolean | false |
| gf.interceptors.loggingZap.zapLoggerEncoding | 日志格式：json 或者 console | string | console |
| gf.interceptors.loggingZap.zapLoggerOutputPaths | 日志文件路径 | []string | stdout |
| gf.interceptors.loggingZap.eventLoggerEncoding | 日志格式：json 或者 console | string | console |
| gf.interceptors.loggingZap.eventLoggerOutputPaths | 日志文件路径 | []string | stdout |

## 获取 RPC 日志实例
每一次 RPC 请求进来的时候，拦截器会把 RequestId（当启动了原数据拦截器）注入到日志实例中。

换句话说，每一个 RPC 请求，都会有一个新的 Logger 实例。我们来看看如何为一个 RPC 请求，记录 ZapLogger 日志。

通过 rkgfctx.GetLogger(ctx) 方法获取本次请求的日志实例。

```go
	gfEntry.Server.BindHandler("/v1/greeter", func(ctx *ghttp.Request) {
		rkgfctx.GetLogger(ctx).Info("Request received")

		ctx.Response.WriteHeader(http.StatusOK)
		ctx.Response.WriteJson(&GreeterResponse{
			Message: fmt.Sprintf("Hello %s!", ctx.GetQuery("name")),
		})
	})
```

日志打印了出来！

```
2021-12-29T14:32:39.164+0800    INFO    eg/main.go:25   Request received
```

## 修改 Event
日志拦截器会为每一个 RPC 请求创建一个 Event 实例。

用户可以添加 pairs，counters，errors。

通过 rkgfctx.GetEvent(ctx) 获取本次 RPC 的 Event 实例。

```go
	gfEntry.Server.BindHandler("/v1/greeter", func(ctx *ghttp.Request) {
		event := rkgfctx.GetEvent(ctx)
		event.AddPair("key", "value")

		ctx.Response.WriteHeader(http.StatusOK)
		ctx.Response.WriteJson(&GreeterResponse{
			Message: fmt.Sprintf("Hello %s!", ctx.GetQuery("name")),
		})
	})
```

Event 里增加了 pairs={"key":"value"}！

```
------------------------------------------------------------------------
endTime=2021-12-29T14:34:04.56917+08:00
startTime=2021-12-29T14:34:04.569026+08:00
elapsedNano=144436
timezone=CST
ids={"eventId":"0387be73-47cb-4394-af7a-6c62d7c72596"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"apiMethod":"GET","apiPath":"/v1/greeter","apiProtocol":"HTTP/1.1","apiQuery":"name=rk-dev","userAgent":"curl/7.64.1"}
error={}
counters={}
pairs={"key":"value"}
timing={}
remoteAddr=localhost:49165
operation=/v1/greeter
resCode=200
eventStatus=Ended
EOE
```
