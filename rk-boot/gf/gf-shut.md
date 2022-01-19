# GoFrame 框架：优雅关闭进程

## 介绍
通过一个完整例子，介绍如何优雅关闭 [gogf/gf](https://github.com/gogf/gf) 微服务。

> **什么是优雅关闭？**
>
> 在进程收到关闭信号时，我们需要关闭后台运行的逻辑，比如，MySQL 连接等等。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 [gogf/gf](https://github.com/gogf/gf) 微服务。
[rk-boot](https://github.com/rookie-ninja/rk-boot) 是一个可通过 YAML 启动多种 Web 服务的框架。请参考本文最后章节，了解 [rk-boot](https://github.com/rookie-ninja/rk-boot) 细节。

请访问如下地址获取完整教程：https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot/gf
```

## 快速开始
### 1.创建 boot.yaml
boot.yaml 文件会告诉 [rk-boot](https://github.com/rookie-ninja/rk-boot) 如何启动 [gogf/gf](https://github.com/gogf/gf) 服务。

```yaml
---
gf:
  - name: greeter
    port: 8080
    enabled: true
```

### 2.创建 main.go
通过 AddShutdownHookFunc() 来添加 shutdownhook 函数。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"fmt"
	"github.com/rookie-ninja/rk-boot"
	_ "github.com/rookie-ninja/rk-boot/gf"
)

func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	boot.AddShutdownHookFunc("shutdown-hook", func() {
		fmt.Println("shutting down")
	})

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.启动 main.go
```
$ go run main.go

2022-01-05T18:24:36.293+0800    INFO    boot/gf_entry.go:1050   Bootstrap gfEntry       {"eventId": "fabd662e-801c-4c8b-b596-f55001a66d62", "entryName": "greeter"}
------------------------------------------------------------------------
endTime=2022-01-05T18:24:36.293501+08:00
startTime=2022-01-05T18:24:36.293309+08:00
elapsedNano=191536
timezone=CST
ids={"eventId":"fabd662e-801c-4c8b-b596-f55001a66d62"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"gfPort":8080}
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

### 4.ctrl-c
通过 ctrl-c 关闭程序，我们会看到打印如下信息。

```
shutting down

2022-01-05T18:24:39.674+0800    INFO    boot/gf_entry.go:1050   Interrupt gfEntry       {"eventId": "f400e32e-6b99-4733-8ec3-1fe186fe8abe", "entryName": "greeter"}
------------------------------------------------------------------------
endTime=2022-01-05T18:24:39.674817+08:00
startTime=2022-01-05T18:24:39.674688+08:00
elapsedNano=128645
timezone=CST
ids={"eventId":"f400e32e-6b99-4733-8ec3-1fe186fe8abe"}
app={"appName":"rk","appVersion":"","entryName":"greeter","entryType":"GfEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"gfPort":8080}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost
operation=Interrupt
resCode=OK
eventStatus=Ended
EOE
```

## rk-boot 介绍
[rk-boot](https://github.com/rookie-ninja/rk-boot) 是一个可通过 YAML 启动多种 Web 服务的框架。
有点类似于 Spring boot。通过集成 rk-xxx 系列库，可以启动多种 Web 框架。当然，用户也可以自定义 rk-xxx 库集成到 rk-boot 中。

![image](img/boot-arch.png)

### rk-boot 亮点
通过同样格式的 YAML 文件，启动不同 Web 框架。

比如，我们可以通过如下文件，在一个进程中同时启动 gRPC, Gin, Echo, GoFrame 框架。统一团队内部的微服务布局。

- 依赖安装

```
go get github.com/rookie-ninja/rk-boot/grpc
go get github.com/rookie-ninja/rk-boot/gin
go get github.com/rookie-ninja/rk-boot/echo
go get github.com/rookie-ninja/rk-boot/gf
```

- boot.yaml 

```
---
grpc:
  - name: grpc-server
    port: 8080
    enabled: true
    commonService:
      enabled: true
gin:
  - name: gin-server
    port: 8081
    enabled: true
    commonService:
      enabled: true
echo:
  - name: echo-server
    port: 8082
    enabled: true
    commonService:
      enabled: true
gf:
  - name: gf-server
    port: 8083
    enabled: true
    commonService:
      enabled: true
```

- main.go

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
	_ "github.com/rookie-ninja/rk-boot/echo"
	_ "github.com/rookie-ninja/rk-boot/gf"
	_ "github.com/rookie-ninja/rk-boot/gin"
	_ "github.com/rookie-ninja/rk-boot/grpc"
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

- 验证

```
# gRPC throuth grpc-gateway
$ curl localhost:8080/rk/v1/healthy
{"healthy":true}

# Gin
$ curl localhost:8081/rk/v1/healthy
{"healthy":true}

# Echo
$ curl localhost:8082/rk/v1/healthy
{"healthy":true}

# GoFrame
$ curl localhost:8083/rk/v1/healthy
{"healthy":true}
```

### rk-boot 支持的 Web 框架
欢迎贡献新的 Web 框架到 rk-boot 系列中。

**参考 [docs](https://rkdev.info/docs/bootstrapper/user-guide/gin-golang/developer/) & [rk-gin](https://github.com/rookie-ninja/rk-gin) 作为例子。**

| 框架 | 开发状态 | 安装 | 依赖 |
| --- | --- | --- | --- |
| [Gin](https://github.com/gin-gonic/gin) | Stable | go get github.com/rookie-ninja/rk-boot/gin | [rk-gin](https://github.com/rookie-ninja/rk-gin) |
| [gRPC](https://grpc.io/)  | Stable | go get github.com/rookie-ninja/rk-boot/grpc | [rk-grpc](https://github.com/rookie-ninja/rk-grpc) |
| [Echo](https://github.com/labstack/echo)  | Stable | go get github.com/rookie-ninja/rk-boot/echo | [rk-echo](https://github.com/rookie-ninja/rk-echo) |
| [GoFrame](https://github.com/gogf/gf)  | Stable | go get github.com/rookie-ninja/rk-boot/gf | [rk-gf](https://github.com/rookie-ninja/rk-gf) |
| [Fiber](https://github.com/gofiber/fiber) | Testing | go get github.com/rookie-ninja/rk-boot/fiber | [rk-fiber](https://github.com/rookie-ninja/rk-fiber) |
| [go-zero](https://github.com/zeromicro/go-zero) | Testing | go get github.com/rookie-ninja/rk-boot/zero | [rk-zero](https://github.com/rookie-ninja/rk-zero) |
| [GorillaMux](https://github.com/gorilla/mux) | Testing | go get github.com/rookie-ninja/rk-boot/mux | [rk-mux](https://github.com/rookie-ninja/rk-mux) |

## rk-gf 介绍
[rk-gf](https://github.com/rookie-ninja/rk-gf) 用于通过 YAML 启动 [gogf/gf](https://github.com/gogf/gf) Web 服务。

![image](img/gf-arch.png)

### 支持的功能
根据 YAML 文件初始化如下的实例，如果是外部以来，均保持原生用法。

| 实例 | 介绍 |
| --- | --- |
| ghttp.Server | 原生 [gogf/gf](https://github.com/gogf/gf) |
| Config | 原生 [spf13/viper](https://github.com/spf13/viper) 参数实例 |
| Logger | 原生 [uber-go/zap](https://github.com/uber-go/zap) 日志实例 |
| EventLogger | 用于记录 RPC 请求日志，使用 [rk-query](https://github.com/rookie-ninja/rk-query) |
| Credential | 用于从远程服务，例如 ETCD 拉取 Credential |
| Cert | 从远程服务（ETCD 等等）中获取 TLS/SSL 证书，并启动 SSL/TLS |
| Prometheus | 启动 Prometheus 客户端，并根据需要推送到 [pushgateway](https://github.com/prometheus/pushgateway) |
| Swagger | 本地启动 Swagger UI |
| CommonService | 暴露通用 API |
| TV | TV 网页，展示微服务的基本信息 |
| StaticFileHandler | 启动 Web 形式的静态文件下载服务，后台存储支持本地文件系统 和 pkger. |

### 支持的中间件
rk-gf 会根据 YAML 文件初始化中间件。

| Middleware | Description |
| --- | --- |
| Metrics | 收集 RPC Metrics，并启动 [prometheus](https://github.com/prometheus/client_golang) |
| Log | 使用 [rk-query](https://github.com/rookie-ninja/rk-query) 记录每一个 RPC 日志 |
| Trace | 收集 RPC 调用链，并且发送数据到 stdout, 本地文件或者 jaeger [open-telemetry/opentelemetry-go](https://github.com/open-telemetry/opentelemetry-go). |
| Panic | Recover from panic for RPC requests and log it. |
| Meta | 收集服务元信息，添加到返回 Header 中 |
| Auth | 支持 [Basic Auth] & [API Key] 验证中间件 |
| RateLimit | RPC 限速中间件 |
| Timeout | RPC 超时中间件 |
| CORS | CORS 中间件 |
| JWT | JWT 验证 |
| Secure | 服务端安全中间件 |
| CSRF | CSRF 中间件 |

### GoFrame 完整 YAML 配置
```yaml
---
#app:
#  description: "this is description"                      # Optional, default: ""
#  keywords: ["rk", "golang"]                              # Optional, default: []
#  homeUrl: "http://example.com"                           # Optional, default: ""
#  iconUrl: "http://example.com"                           # Optional, default: ""
#  docsUrl: ["http://example.com"]                         # Optional, default: []
#  maintainers: ["rk-dev"]                                 # Optional, default: []
#zapLogger:
#  - name: zap-logger                                      # Required
#    description: "Description of entry"                   # Optional
#eventLogger:
#  - name: event-logger                                    # Required
#    description: "Description of entry"                   # Optional
#cred:
#  - name: "local-cred"                                    # Required
#    provider: "localFs"                                   # Required, etcd, consul, localFs, remoteFs are supported options
#    description: "Description of entry"                   # Optional
#    locale: "*::*::*::*"                                  # Optional, default: *::*::*::*
#    paths:                                                # Optional
#      - "example/boot/full/cred.yaml"
#cert:
#  - name: "local-cert"                                    # Required
#    provider: "localFs"                                   # Required, etcd, consul, localFs, remoteFs are supported options
#    description: "Description of entry"                   # Optional
#    locale: "*::*::*::*"                                  # Optional, default: *::*::*::*
#    serverCertPath: "example/boot/full/server.pem"        # Optional, default: "", path of certificate on local FS
#    serverKeyPath: "example/boot/full/server-key.pem"     # Optional, default: "", path of certificate on local FS
#    clientCertPath: "example/client.pem"                  # Optional, default: "", path of certificate on local FS
#    clientKeyPath: "example/client.pem"                   # Optional, default: "", path of certificate on local FS
#config:
#  - name: rk-main                                         # Required
#    path: "example/boot/full/config.yaml"                 # Required
#    locale: "*::*::*::*"                                  # Required, default: *::*::*::*
#    description: "Description of entry"                   # Optional
gf:
  - name: greeter                                          # Required
    port: 8080                                             # Required
    enabled: true                                          # Required
#    description: "greeter server"                         # Optional, default: ""
#    cert:
#      ref: "local-cert"                                   # Optional, default: "", reference of cert entry declared above
#    sw:
#      enabled: true                                       # Optional, default: false
#      path: "sw"                                          # Optional, default: "sw"
#      jsonPath: ""                                        # Optional
#      headers: ["sw:rk"]                                  # Optional, default: []
#    commonService:
#      enabled: true                                       # Optional, default: false
#    static:
#      enabled: true                                       # Optional, default: false
#      path: "/rk/v1/static"                               # Optional, default: /rk/v1/static
#      sourceType: local                                   # Required, options: pkger, local
#      sourcePath: "."                                     # Required, full path of source directory
#    tv:
#      enabled:  true                                      # Optional, default: false
#    prom:
#      enabled: true                                       # Optional, default: false
#      path: ""                                            # Optional, default: "metrics"
#      pusher:
#        enabled: false                                    # Optional, default: false
#        jobName: "greeter-pusher"                         # Required
#        remoteAddress: "localhost:9091"                   # Required
#        basicAuth: "user:pass"                            # Optional, default: ""
#        intervalMs: 10000                                 # Optional, default: 1000
#        cert:                                             # Optional
#          ref: "local-test"                               # Optional, default: "", reference of cert entry declared above
#    logger:
#      zapLogger:
#        ref: zap-logger                                   # Optional, default: logger of STDOUT, reference of logger entry declared above
#      eventLogger:
#        ref: event-logger                                 # Optional, default: logger of STDOUT, reference of logger entry declared above
#    interceptors:
#      loggingZap:
#        enabled: true                                     # Optional, default: false
#        zapLoggerEncoding: "json"                         # Optional, default: "console"
#        zapLoggerOutputPaths: ["logs/app.log"]            # Optional, default: ["stdout"]
#        eventLoggerEncoding: "json"                       # Optional, default: "console"
#        eventLoggerOutputPaths: ["logs/event.log"]        # Optional, default: ["stdout"]
#      metricsProm:
#        enabled: true                                     # Optional, default: false
#      auth:
#        enabled: true                                     # Optional, default: false
#        basic:
#          - "user:pass"                                   # Optional, default: []
#        ignorePrefix:
#          - "/rk/v1"                                      # Optional, default: []
#        apiKey:
#          - "keys"                                        # Optional, default: []
#      meta:
#        enabled: true                                     # Optional, default: false
#        prefix: "rk"                                      # Optional, default: "rk"
#      tracingTelemetry:
#        enabled: true                                     # Optional, default: false
#        exporter:                                         # Optional, default will create a stdout exporter
#          file:
#            enabled: true                                 # Optional, default: false
#            outputPath: "logs/trace.log"                  # Optional, default: stdout
#          jaeger:
#            agent:
#              enabled: false                              # Optional, default: false
#              host: ""                                    # Optional, default: localhost
#              port: 0                                     # Optional, default: 6831
#            collector:
#              enabled: true                               # Optional, default: false
#              endpoint: ""                                # Optional, default: http://localhost:14268/api/traces
#              username: ""                                # Optional, default: ""
#              password: ""                                # Optional, default: ""
#      rateLimit:
#        enabled: false                                    # Optional, default: false
#        algorithm: "leakyBucket"                          # Optional, default: "tokenBucket"
#        reqPerSec: 100                                    # Optional, default: 1000000
#        paths:
#          - path: "/rk/v1/healthy"                        # Optional, default: ""
#            reqPerSec: 0                                  # Optional, default: 1000000
#      jwt:
#        enabled: true                                     # Optional, default: false
#        signingKey: "my-secret"                           # Required
#        ignorePrefix:                                     # Optional, default: []
#          - "/rk/v1/tv"
#          - "/sw"
#          - "/rk/v1/assets"
#        signingKeys:                                      # Optional
#          - "key:value"
#        signingAlgo: ""                                   # Optional, default: "HS256"
#        tokenLookup: "header:<name>"                      # Optional, default: "header:Authorization"
#        authScheme: "Bearer"                              # Optional, default: "Bearer"
#      secure:
#        enabled: true                                     # Optional, default: false
#        xssProtection: ""                                 # Optional, default: "1; mode=block"
#        contentTypeNosniff: ""                            # Optional, default: nosniff
#        xFrameOptions: ""                                 # Optional, default: SAMEORIGIN
#        hstsMaxAge: 0                                     # Optional, default: 0
#        hstsExcludeSubdomains: false                      # Optional, default: false
#        hstsPreloadEnabled: false                         # Optional, default: false
#        contentSecurityPolicy: ""                         # Optional, default: ""
#        cspReportOnly: false                              # Optional, default: false
#        referrerPolicy: ""                                # Optional, default: ""
#        ignorePrefix: []                                  # Optional, default: []
#      csrf:
#        enabled: true
#        tokenLength: 32                                   # Optional, default: 32
#        tokenLookup: "header:X-CSRF-Token"                # Optional, default: "header:X-CSRF-Token"
#        cookieName: "_csrf"                               # Optional, default: _csrf
#        cookieDomain: ""                                  # Optional, default: ""
#        cookiePath: ""                                    # Optional, default: ""
#        cookieMaxAge: 86400                               # Optional, default: 86400
#        cookieHttpOnly: false                             # Optional, default: false
#        cookieSameSite: "default"                         # Optional, default: "default", options: lax, strict, none, default
#        ignorePrefix: []                                  # Optional, default: []
```
