# gRPC: How to add logging interceptor/middleware in gRPC?

## Introduction
Add logging interceptor/middleware in gRPC micro-service easily with [rk-boot](https://github.com/rookie-ninja/rk-boot) and [rk-grpc](https://github.com/rookie-ninja/rk-grpc).

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

[rk-boot](https://github.com/rookie-ninja/rk-boot) integrate two open source libraries while logging gRPC API.
 
- [uber-go/zap](https://github.com/uber-go/zap) as logging base.
- [logrus](https://github.com/sirupsen/logrus) as log rotation.

### 1.Create boot.yaml
In order to validate logging, we enable [CommonService](https://github.com/rookie-ninja/rk-grpc#common-service-1) which contains some commonly used APIs.

rk-boot will enable grpc-gateway by default. As a result, we can send restful request directly to gRPC server.

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

### 2.Create main.go 
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

### 3.Directory hierarchy
```
$ tree
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

### 4.Validate
```
$ go run main.go
```

- Send request

```
$ curl -X GET localhost:8080/rk/v1/healthy
{"healthy":true}
```

- Validate log
The log format comes from [rk-query](https://github.com/rookie-ninja/rk-query) which support both console and json types of encoding.

We will describe later.

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

## Change log format
We can change value of eventLoggerEncoding in boot.yaml.

json and console are supported values currently.

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

### Change log path
We can change value of eventLoggerOutputPaths in boot.yaml.

By default, logs will be rotated by 1GB of file size.

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

## Concept
We introduce two types of logger.
- ZapLogger
- EventLogger

### ZapLogger
ZapLogger is used to log errors or custom information by user. rk-boot will create a new ZapLogger instance for every RPC request with RequestId in it.

```
2021-07-09T23:50:39.318+0800    INFO    basic/main.go:36        Received request        {"requestId": "c33698f2-3071-48d4-9d92-b1aa311e6c06"}
```

### EventLogger
rk-boot defines RPC request as an Event, and record every RPC request into Event type in rk-query.

| Field | Description |
| ---- | ---- |
| endTime | As name described |
| startTime | As name described |
| elapsedNano | Elapsed time for RPC in nanoseconds |
| timezone | As name described |
| ids | Contains three different ids(eventId, requestId and traceId). If meta interceptor was enabled or event.SetRequestId() was called by user, then requestId would be attached. eventId would be the same as requestId if meta interceptor was enabled. If trace interceptor was enabled, then traceId would be attached. |
| app | Contains [appName, appVersion](https://github.com/rookie-ninja/rk-entry#appinfoentry), entryName, entryType. |
| env | Contains arch, az, domain, hostname, localIP, os, realm, region. realm, region, az, domain were retrieved from environment variable named as REALM, REGION, AZ and DOMAIN. "*" means empty environment variable.|
| payloads | Contains RPC related metadata |
| error | Contains errors if occur |
| counters | Set by calling event.SetCounter() by user. |
| pairs | Set by calling event.AddPair() by user. |
| timing | Set by calling event.StartTimer() and event.EndTimer() by user. |
| remoteAddr |  As name described |
| operation | RPC method name |
| resCode | Response code of RPC |
| eventStatus | Ended or InProgress |

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

## Log interceptor options
| name | description | type | default value |
| ------ | ------ | ------ | ------ |
| grpc.interceptors.loggingZap.enabled | Enable log interceptor | boolean | false |
| grpc.interceptors.loggingZap.zapLoggerEncoding | json or console | string | console |
| grpc.interceptors.loggingZap.zapLoggerOutputPaths | Output paths | []string | stdout |
| grpc.interceptors.loggingZap.eventLoggerEncoding | json or console | string | console |
| grpc.interceptors.loggingZap.eventLoggerOutputPaths | Output paths | []string | stdout |

## Get RPC scope logger
A new zap logger instance with requestId(if exist enabled by meta interceptor) will be created for every RPC request.

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	rkgrpcctx.GetLogger(ctx).Info("Received request")

	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

Log with requestId will be printed!

```
2021-07-09T23:50:39.318+0800    INFO    basic/main.go:36        Received request        {"requestId": "c33698f2-3071-48d4-9d92-b1aa311e6c06"}
```

## Add values into Event
A new event logger instance will be created for every RPC request.

User can add pairs, counters, errors in event which will be logged as soon as RPC finish.

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	event := rkgrpcctx.GetEvent(ctx)
	event.AddPair("key", "value")

	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

pairs={"key":"value"} will be printed!

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





