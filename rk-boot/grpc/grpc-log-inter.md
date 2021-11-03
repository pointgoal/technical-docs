# GRPC: 如何添加 API 日志拦截器/中间件？

## 介绍

本文将介绍如何在 gRPC 微服务中添加 API 日志拦截器/中间件。

> 什么是日志拦截器/中间件？
>
> 日志拦截器会对每一个 API 请求记录日志。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 gRPC 服务。

> 请访问如下地址获取完整教程：
> - https://rkdev.info/cn
> - https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
[rk-boot](https://github.com/rookie-ninja/rk-boot) 默认集成如下两个开源库。
 
- [uber-go/zap](https://github.com/uber-go/zap) 作为底层的日志库。
- [logrus](https://github.com/sirupsen/logrus) 作为日志滚动。

### 1.创建 boot.yaml
为了验证，我们同时启动了 commonService。commonService 里包含了一系列通用 API。
详情: [CommonService](https://github.com/rookie-ninja/rk-grpc#common-service-1)

grpc 默认会启动 grpc-gateway 来提供 Restful API 服务。在验证的时候，我们可以直接发送 Restful 请求。

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    commonService:
      enabled: true                 # Enable common service for testing
    interceptors:
      loggingZap:
        enabled: true               # Enable logging interceptor
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

### 3.验证
```
$ go run main.go
```

- 发送请求

```
$ curl -X GET localhost:8080/rk/v1/healthy
{"healthy":true}
```

- 验证日志
默认输出到 stdout（控制台）。

下面的日志格式来自 [rk-query](https://github.com/rookie-ninja/rk-query) ，用户也可以选择 JSON 格式，我们稍后会介绍。

```
------------------------------------------------------------------------
endTime=2021-07-09T23:44:09.81483+08:00
startTime=2021-07-09T23:44:09.814784+08:00
elapsedNano=46065
timezone=CST
ids={"eventId":"67d64dab-f3ea-4b77-93d0-6782caf4cfee"}
app={"appName":"rk-demo","appVersion":"master-f414049","entryName":"greeter","entryType":"GrpcEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Healthy","grpcService":"rk.api.v1.RkCommonService","grpcType":"unaryServer","gwMethod":"","gwPath":"","gwScheme":"","gwUserAgent":""}
error={}
counters={}
pairs={"healthy":"true"}
timing={}
remoteAddr=localhost:58205
operation=/rk.api.v1.RkCommonService/Healthy
resCode=OK
eventStatus=Ended
EOE
```

## 修改日志格式
我们可以通过修改 boot.yaml 来修改日志格式。
目前支持 json 和 console 两种格式，默认为 console。

通过修改 eventLoggerEncoding 的值为 json，我们可以把日志的输出为 JSON 格式。

```
grpc:
  - name: greeter                     # Name of grpc entry
    port: 8080                        # Port of grpc entry
    enabled: true                     # Enable grpc entry
    commonService:
      enabled: true                   # Enable common service for testing
    interceptors:
      loggingZap:
        enabled: true                 # Enable logging interceptor
        zapLoggerEncoding: "json"     # Override to json format, option: json or console
        eventLoggerEncoding: "json"   # Override to json format, option: json or console
```

## 修改日志路径
通过修改 eventLoggerOutputPaths 的值，可以指定输出路径。

日志默认在 1GB 之后，进行切割，并压缩。

```
grpc:
  - name: greeter                     # Name of grpc entry
    port: 8080                        # Port of grpc entry
    enabled: true                     # Enable grpc entry
    commonService:
      enabled: true                   # Enable common service for testing
    interceptors:
      loggingZap:
        enabled: true                 # Enable logging interceptor
        zapLoggerOutputPaths: ["logs/app.log"]        # Override output paths, option: json or console
        eventLoggerOutputPaths: ["logs/event.log"]    # Override output paths, option: json or console
```

## 概念
验证了日志拦截器，我们来具体讲一下 rk-boot 提供的日志拦截器都有哪些功能。

我们需要提前了解两个概念。
- EventLogger
- ZapLogger

### ZapLogger
用于记录错误/详细日志，用户可以获取本次 RPC 调用的 ZapLogger 实例，进行日志写入，每个 RPC 的 ZapLogger 实例都包含当前的 RequestId。

```
2021-07-09T23:52:13.667+0800    INFO    boot/grpc_entry.go:694  Bootstrapping grpcEntry.        {"eventId": "9bc192fb-567c-45d4-8775-7a097b0dab04", "entryName": "greeter", "entryType": "GrpcEntry", "grpcPort": 8080, "commonServiceEnabled": true, "tlsEnabled": false, "gwEnabled": true, "reflectionEnabled": false, "swEnabled": false, "tvEnabled": false, "promEnabled": false, "gwClientTlsEnabled": false, "gwServerTlsEnabled": false}
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
endTime=2021-07-09T23:44:09.81483+08:00
startTime=2021-07-09T23:44:09.814784+08:00
elapsedNano=46065
timezone=CST
ids={"eventId":"67d64dab-f3ea-4b77-93d0-6782caf4cfee"}
app={"appName":"rk-demo","appVersion":"master-f414049","entryName":"greeter","entryType":"GrpcEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Healthy","grpcService":"rk.api.v1.RkCommonService","grpcType":"unaryServer","gwMethod":"","gwPath":"","gwScheme":"","gwUserAgent":""}
error={}
counters={}
pairs={"healthy":"true"}
timing={}
remoteAddr=localhost:58205
operation=/rk.api.v1.RkCommonService/Healthy
resCode=OK
eventStatus=Ended
EOE
```

## 日志拦截器选项
| 名字 | 描述 | 类型 | 默认值 |
| ------ | ------ | ------ | ------ |
| grpc.interceptors.loggingZap.enabled | 启动日志拦截器 | boolean | false |
| grpc.interceptors.loggingZap.zapLoggerEncoding | 日志格式：json 或者 console | string | console |
| grpc.interceptors.loggingZap.zapLoggerOutputPaths | 日志文件路径 | []string | stdout |
| grpc.interceptors.loggingZap.eventLoggerEncoding | 日志格式：json 或者 console | string | console |
| grpc.interceptors.loggingZap.eventLoggerOutputPaths | 日志文件路径 | []string | stdout |

## 获取 RPC 日志实例
每一次 RPC 请求进来的时候，拦截器会把 RequestId（当启动了原数据拦截器）注入到日志实例中。

换句话说，每一个 RPC 请求，都会有一个新的 Logger 实例。我们来看看如何为一个 RPC 请求，记录 ZapLogger 日志。

通过 rkgrpcctx.GetLogger(ctx) 方法获取本次请求的日志实例。

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	rkgrpcctx.GetLogger(ctx).Info("Received request")

	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

日志打印了出来！

```
2021-07-09T23:50:39.318+0800    INFO    basic/main.go:36        Received request        {"requestId": "c33698f2-3071-48d4-9d92-b1aa311e6c06"}
```

## 修改 Event
日志拦截器会为每一个 RPC 请求创建一个 Event 实例。

用户可以添加 pairs，counters，errors。

通过 rkgrpcctx.GetEvent(ctx) 获取本次 RPC 的 Event 实例。

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	event := rkgrpcctx.GetEvent(ctx)
	event.AddPair("key", "value")

	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

Event 里增加了 pairs={"key":"value"}！

```
------------------------------------------------------------------------
endTime=2021-07-09T23:52:39.103351+08:00
startTime=2021-07-09T23:52:39.10332+08:00
elapsedNano=31154
timezone=CST
ids={"eventId":"92001951-80c1-4dda-8f14-f920834f5c61","requestId":"92001951-80c1-4dda-8f14-f920834f5c61"}
app={"appName":"rk-demo","appVersion":"master-f414049","entryName":"greeter","entryType":"GrpcEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Greeter","grpcService":"api.v1.Greeter","grpcType":"unaryServer","gwMethod":"","gwPath":"","gwScheme":"","gwUserAgent":""}
error={}
counters={}
pairs={"key":"value"}
timing={}
remoteAddr=localhost:58269
operation=/api.v1.Greeter/Greeter
resCode=OK
eventStatus=Ended
EOE
```


