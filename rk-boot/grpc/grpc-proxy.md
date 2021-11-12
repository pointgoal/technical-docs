# GRPC: 实现 gRPC 代理

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 快速搭建 gRPC 代理。

> **什么是 gRPC 代理？** 
>
> gRPC 代理会接受 gRPC 请求，并根据用户策略转发至其他 gRPC 服务。应用场景不多，比如根据环境参数，把请求转发到不同的 gRPC 服务。

**请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 启动的 gRPC 代理有一个限制。只有通过代码形式发送的请求，才可以被代理。grpc-gateway 或者 grpcurl 形式的请求暂时不支持。

目前，rk-boot 支持3中策略。

- **headerBased:** 通过 gRPC 请求里的 Metadata 值来判断代理目的地。
- **pathBased:** 通过请求路径来判断代理目的地。
- **ipBased:** 通过远程 IP 地址来判断代理目的地。

在下面的例子中，我们启动三个服务。

- proxy（8080）: 代理服务，根据 headerBased 策略，转发 rk.api.v1.RkCommonService.Healthy 请求到 test 服务中。
- test（8081）: 测试域 gRPC 服务，接受 proxy 代理过来的请求。
- client: 往 proxy 服务发送 rk.api.v1.RkCommonService.Healthy 请求。

### 1.创建 proxy/boot.yaml & proxy/main.go
监听 8080 端口，proxy 服务没有实现任何 gRPC 方法，如果 gRPC 请求的 Metadata 中包含 domain:test，会转发。

代理会默认从 proxy.rules.dest 中挑选一个地址转发。

- boot.yaml

```yaml
---
grpc:
  - name: greeter                        # Required
    port: 8080                           # Required
    enabled: true                        # Required
    proxy:
      enabled: true                      # Optional, enable proxy server
      rules:
        - type: headerBased              # Optional, options:[headerBased, pathBased, ipBased]
          headerPairs: ["domain:test"]   # Optional, header pairs separated by colon(:)
          dest: ["localhost:8081"]       # Optional, destinations
    interceptors:
      loggingZap:
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

### 2.创建 test/boot.yaml & test/main.go
监听 8081 端口。

启动 CommonService 服务，接收 Proxy 代理过来的 rk.api.v1.RkCommonService.Healthy 请求。

- boot.yaml

```yaml
---
grpc:
  - name: greeter                     # Required
    port: 8081                        # Required
    enabled: true                     # Required
    commonService:
      enabled: true                   # Optional, default: false
    interceptors:
      loggingZap:
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

### 3.client/main.go
发送的请求 metadata 里，添加 domain:test，让 proxy 代理请求。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"fmt"
	api "github.com/rookie-ninja/rk-grpc/boot/api/third_party/gen/v1"
	"github.com/rookie-ninja/rk-grpc/interceptor/context"
	"github.com/rookie-ninja/rk-grpc/interceptor/log/zap"
	"go.uber.org/zap"
	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"log"
)

// In this example, we will create a simple gRpc client and enable RK style logging interceptor.
func main() {
	// ********************************************
	// ********** Enable interceptors *************
	// ********************************************
	opts := []grpc.DialOption{
		grpc.WithChainUnaryInterceptor(
			rkgrpclog.UnaryClientInterceptor(),
		),
		grpc.WithInsecure(),
		grpc.WithBlock(),
	}

	// 1: Create grpc client
	conn, client := createCommonServiceClient(opts...)
	defer conn.Close()

	// 2: Wrap context, this is required in order to use bellow features easily.
	ctx := rkgrpcctx.WrapContext(context.Background())
	// Add header to make proxy request to test server
	ctx = metadata.AppendToOutgoingContext(ctx, "domain", "test")

	// 3: Call server
	if resp, err := client.Healthy(ctx, &api.HealthyRequest{}); err != nil {
		rkgrpcctx.GetLogger(ctx).Fatal("Failed to send request to server.", zap.Error(err))
	} else {
		rkgrpcctx.GetLogger(ctx).Info(fmt.Sprintf("[Message]: %s", resp.String()))
	}
}

func createCommonServiceClient(opts ...grpc.DialOption) (*grpc.ClientConn, api.RkCommonServiceClient) {
	// 1: Set up a connection to the server.
	conn, err := grpc.DialContext(context.Background(), "localhost:8080", opts...)
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}

	// 2: Create grpc client
	client := api.NewRkCommonServiceClient(conn)

	return conn, client
}
```

### 4.文件夹结构 
```
$ tree
.
├── client
│   ├── go.mod
│   └── main.go
├── proxy
│   ├── boot.yaml
│   ├── go.mod
│   ├── go.sum
│   └── main.go
└── test
    ├── boot.yaml
    ├── go.mod
    └── main.go

3 directories, 9 files
```

### 4.启动 proxy，test
```
$ go run proxy/main.go
$ go run test/main.go
```

### 6.验证
启动 client/main.go

- client 端日志

```
2021-11-13T02:14:33.598+0800    INFO    client/main.go:45       [Message]: fields:{key:"healthy" value:{bool_value:true}}

------------------------------------------------------------------------
endTime=2021-11-13T02:14:33.597875+08:00
startTime=2021-11-13T02:14:33.59247+08:00
elapsedNano=5405233
timezone=CST
ids={"eventId":"7ffe293d-24f4-42b3-9325-64bde414912c"}
app={"appName":"rk","appVersion":"","entryName":"grpc","entryType":"grpc"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Healthy","grpcService":"rk.api.v1.RkCommonService","grpcType":"unaryClient","remoteIp":"localhost","remotePort":"8080"}
error={}
counters={}
pairs={}
timing={}
remoteAddr=localhost:8080
operation=/rk.api.v1.RkCommonService/Healthy
resCode=OK
eventStatus=Ended
EOE
```

- proxy 端日志

proxy 端的 grpcType 是 streamServer。

```
------------------------------------------------------------------------
endTime=2021-11-13T02:14:33.597337+08:00
startTime=2021-11-13T02:14:33.593252+08:00
elapsedNano=4085300
timezone=CST
ids={"eventId":"f32f7895-a4b2-4c8c-bedc-1d4733db78a7"}
app={"appName":"rk-demo","appVersion":"master-2c9c6fd","entryName":"greeter","entryType":"GrpcEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Healthy","grpcService":"rk.api.v1.RkCommonService","grpcType":"streamServer","gwMethod":"","gwPath":"","gwScheme":"","gwUserAgent":""}
error={}
counters={}
pairs={}
timing={}
remoteAddr=127.0.0.1:58441
operation=/rk.api.v1.RkCommonService/Healthy
resCode=OK
eventStatus=Ended
EOE
```

- test 端日志

test 端的 grpcType 是 unaryServer。

```
------------------------------------------------------------------------
endTime=2021-11-13T02:14:33.596967+08:00
startTime=2021-11-13T02:14:33.59692+08:00
elapsedNano=47149
timezone=CST
ids={"eventId":"6eb9b10a-153a-4955-85b0-f227e3ec54b2"}
app={"appName":"rk-demo","appVersion":"master-2c9c6fd","entryName":"greeter","entryType":"GrpcEntry"}
env={"arch":"amd64","az":"*","domain":"*","hostname":"lark.local","localIP":"10.8.0.2","os":"darwin","realm":"*","region":"*"}
payloads={"grpcMethod":"Healthy","grpcService":"rk.api.v1.RkCommonService","grpcType":"unaryServer","gwMethod":"","gwPath":"","gwScheme":"","gwUserAgent":""}
error={}
counters={}
pairs={"healthy":"true"}
timing={}
remoteAddr=10.8.0.2:58443
operation=/rk.api.v1.RkCommonService/Healthy
resCode=OK
eventStatus=Ended
EOE
```

## pathBased 策略
可以通过修改 boot.yaml 文件修改代理策略。

```yaml
---
grpc:
  - name: greeter                                         # Required
    port: 8080                                            # Required
    enabled: true                                         # Required
    proxy:
      enabled: true                                       # Optional, enable proxy server
      rules:
        - type: pathBased                                 # Optional, options:[headerBased, pathBased, ipBased]
          paths: ["/rk.api.v1.RkCommonService/Healthy"]   # Optional, gRPC method, support golang regex.
          dest: ["localhost:8081"]                        # Optional, destinations
```

## ipBased 策略
可以通过修改 boot.yaml 文件修改代理策略。

proxy.rules.ips 支持 CIDR。

```yaml
---
grpc:
  - name: greeter                    # Required
    port: 8080                       # Required
    enabled: true                    # Required
    proxy:
      enabled: true                  # Optional, enable proxy server
      rules:
        - type: ipBased              # Optional, options:[headerBased, pathBased, ipBased]
          ips: ["127.0.0.0/24"]      # Optional, remote Ips, support CIDR.
          dest: ["localhost:8081"]   # Optional, destinations
```
