# GRPC: 如何让 gRPC 提供 Restful API 服务?

## 介绍
本文将介绍如何让一个 gRPC 服务，同时提供 gRPC 和 Restful API。

- 为了能让 gRPC 提供 REST API，我们需要使用 [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)

## 使用 rk-boot
[rk-boot](https://github.com/rookie-ninja/rk-boot) 是集成了 Gin, gRPC 和一系列流行 Go 语言框架的启动器，用户可以通过 rk-boot 快速启动企业级 Go 语言微服务。

### 先决条件
使用过 GRPC 的用户都应该知道，protocol buffer 文件需要使用相关的命令行，把 *.proto 文件编译成 *.go 文件。

根据不同需要，会使用到不同的命令行文件。以 Go 语言为例，我们需要大致如下几个命令行文件。

| 工具 | 介绍 | 安装 |
| ---- | ---- | ---- |
| [protobuf](https://github.com/protocolbuffers/protobuf) | protocol buffer 编译所需的命令行 | [Install](http://google.github.io/proto-lens/installing-protoc.html) |
| [protoc-gen-go](https://github.com/golang/protobuf/tree/master/protoc-gen-go) | 从 proto 文件，生成 .go 文件 | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-go-grpc](https://github.com/grpc/grpc-go) | 从 proto 文件，生成 GRPC 相关的 .go 文件 | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) | 从 proto 文件，生成 grpc-gateway 相关的 .go 文件 | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |
| [protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway) | 从 proto 文件，生成 swagger 界面所需的参数文件 | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |

除了安装上述命令行，我们还需要根据需要，运行至少4种不同命令来编译 *.proto 文件，非常晦涩难懂。

> 具体操作方式可参考我的前一篇文章：【GRPC: 使用 Buf 快速编译 GRPC proto 文件】
> 或者访问：【https://rkdev.info/cn/docs/bootstrapper/user-guide/grpc-golang/basic/grpc-gateway/】

### 安装
```go
go get github.com/rookie-ninja/rk-boot
```

### 快速开始
#### 1.创建 api/v1/greeter.proto 
```
syntax = "proto3";

package api.v1;

option go_package = "api/v1/greeter";

service Greeter {
  rpc Greeter (GreeterRequest) returns (GreeterResponse) {}
}

message GreeterRequest {
  string name = 1;
}

message GreeterResponse {
  string message = 1;
}
```

#### 2.创建 api/v1/gw_mapping.yaml 
```
type: google.api.Service
config_version: 3

# Please refer google.api.Http in https://github.com/googleapis/googleapis/blob/master/google/api/http.proto file for details.
http:
  rules:
    - selector: api.v1.Greeter.Greeter
      get: /api/v1/greeter
```

#### 3.创建 buf.yaml
```
version: v1beta1
name: github.com/rk-dev/rk-demo
build:
  roots:
    - api
```

#### 4.创建 buf.gen.yaml
```
version: v1beta1
plugins:
  # protoc-gen-go needs to be installed, generate go files based on proto files
  - name: go
    out: api/gen
    opt:
     - paths=source_relative
  # protoc-gen-go-grpc needs to be installed, generate grpc go files based on proto files
  - name: go-grpc
    out: api/gen
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
  # protoc-gen-grpc-gateway needs to be installed, generate grpc-gateway go files based on proto files
  - name: grpc-gateway
    out: api/gen
    opt:
      - paths=source_relative
      - grpc_api_configuration=api/v1/gw_mapping.yaml
  # protoc-gen-openapiv2 needs to be installed, generate swagger config files based on proto files
  - name: openapiv2
    out: api/gen
    opt:
      - grpc_api_configuration=api/v1/gw_mapping.yaml
```

#### 5.编译 proto file 
```
$ buf generate
```

> 如下的文件会被创建。

```
$ tree api/gen 
api/gen
└── v1
    ├── greeter.pb.go
    ├── greeter.pb.gw.go
    ├── greeter.swagger.json
    └── greeter_grpc.pb.go
 
1 directory, 4 files
```

#### 6.创建 boot.yaml
```
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    enableReflection: true
```

#### 7. 创建 main.go
```
package main

import (
	"context"
	"fmt"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-demo/api/gen/v1"
	"google.golang.org/grpc"
)

// Application entrance.
func main() {
    // Create a new boot instance.
    boot := rkboot.NewBoot()

    // ***************************************
    // ******* Register GRPC & Gateway *******
    // ***************************************

    // Get grpc entry with name
    grpcEntry := boot.GetGrpcEntry("greeter")
    // Register grpc registration function
    grpcEntry.AddRegFuncGrpc(registerGreeter)
    // Register grpc-gateway registration function
    grpcEntry.AddRegFuncGw(greeter.RegisterGreeterHandlerFromEndpoint)

    // Bootstrap
    boot.Bootstrap(context.Background())

    // Wait for shutdown sig
    boot.WaitForShutdownSig(context.Background())
}

// Implementation of [type GrpcRegFunc func(server *grpc.Server)]
func registerGreeter(server *grpc.Server) {
    greeter.RegisterGreeterServer(server, &GreeterServer{})
}

// Implementation of grpc service defined in proto file
type GreeterServer struct{}

func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
    return &greeter.GreeterResponse{
        Message: fmt.Sprintf("Hello %s!", request.Name),
    }, nil
}
```

#### 8.文件夹结构 
```
$ tree
.
├── api
│   ├── gen
│   │   └── v1
│   │       ├── greeter.pb.go
│   │       ├── greeter.pb.gw.go
│   │       ├── greeter.swagger.json
│   │       └── greeter_grpc.pb.go
│   └── v1
│       ├── greeter.proto
│       └── gw_mapping.yaml
├── boot.yaml
├── buf.gen.yaml
├── buf.yaml
├── go.mod
├── go.sum
└── main.go

4 directories, 12 files
```

#### 9.验证
> rk-boot 会使用同一个端口来映射 gRPC 和 gRPC-gateway，非常方便。

```
$ go run main.go
```

- 验证 Restful API

```
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
{"message":"Hello rk-dev!"}
```

- 验证 gRPC
```
$ grpcurl -d '{"name": "rk-dev"}' -plaintext localhost:8080 api.v1.Greeter.Greeter
{
  "message": "Hello rk-dev!"
}
```

### Gateway server option
rk-boot 有一个 enableRkGwOption 选项，启动之后，会包含如下几个功能。

> RK 推荐的 server option，提供如下的功能。
>
> [rkgrpc.RkGwServerMuxOptions](https://github.com/rookie-ninja/rk-grpc/blob/master/boot/gw_server_options.go)

| 功能 | 详情 |
| ---- | ---- |
| HttpErrorHandler | 主要代码从原有 grpc-gateway 代码中抄写而来，启动器会返回 RK 推荐的 API 错误结构 |
| MarshalerOption | protojson.MarshalOptions{UseProtoNames: false, EmitUnpopulated: true},UnmarshalOptions: protojson.UnmarshalOptions{}} |
| Metadata | 注入如下 GRPC metadata： x-forwarded-method, x-forwarded-path, x-forwarded-scheme, x-forwarded-user-agent 和 x-forwarded-remote-addr |
| OutgoingHeaderMatcher | 把原有的 grpc metadata 转发到 http 头部（没有前缀） |
| IncomingHeaderMatcher | 把原有的 http 头部 转发到 grpc metadata（没有前缀） |


#### 1.启动 enableRkGwOption
为了能够验证，我们会添加 gRPC 日志拦截器。

- boot.yaml

```
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    enableReflection: true          # Enable grpc reflection
    enableRkGwOption: true          # Enable rk style grpc gateway server option
    interceptors:
      loggingZap:
        enabled: true               # Enable logging interceptor for validation
```

#### 2.验证日志
```
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
```

> gwMethod, gwPath, gwScheme, gwUserAgent 将会记录到日志中
 
```
------------------------------------------------------------------------
endTime=2021-07-09T21:03:43.518106+08:00
...
payloads={"grpcMethod":"Greeter","grpcService":"api.v1.Greeter","grpcType":"unaryServer","gwMethod":"GET","gwPath":"/v1/greeter","gwScheme":"http","gwUserAgent":"curl/7.64.1"}
...
```

#### 3.验证传入的 metadata 
```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
    // Print incoming headers
    fmt.Println(rkgrpcctx.GetIncomingHeaders(ctx))

    return &greeter.GreeterResponse{
        Message: fmt.Sprintf("Hello %s!", request.Name),
    }, nil
}
```

```
map[:authority:[0.0.0.0:8080] accept:[*/*] content-type:[application/grpc] user-agent:[grpc-go/1.38.0] x-forwarded-for:[::1] x-forwarded-host:[localhost:8080] x-forwarded-method:[GET] x-forwarded-path:[/v1/greeter] x-forwarded-remote-addr:[[::1]:57082] x-forwarded-scheme:[http] x-forwarded-user-agent:[curl/7.64.1]]
```

### 错误映射
grpc-gateway 中，我们需要了解 gateway 到 grpc 的错误映射，即 http 到 grpc 的错误映射。

这是默认的 grpc-gateway 中的错误映射。

| GRPC 错误码 | GRPC 错误码描述 | Gateway(Http) 错误码 | Gateway(Http) 错误码描述 |
| ---- | ---- | ---- | ---- |
| 0 | OK | 200 | OK |
| 1 | CANCELLED | 408 | Request Timeout |
| 2 | UNKNOWN | 500 | Internal Server Error |
| 3 | INVALID_ARGUMENT | 400 | Bad Request |
| 4 | DEADLINE_EXCEEDED | 504 | Gateway Timeout |
| 5 | NOT_FOUND | 404 | Not Found |
| 6 | ALREADY_EXISTS | 409 | Conflict |
| 7 | PERMISSION_DENIED | 403 | Forbidden |
| 8 | RESOURCE_EXHAUSTED | 429 | Too Many Requests |
| 9 | FAILED_PRECONDITION | 400 | Bad Request |
| 10 | ABORTED | 409 | Conflict |
| 11 | OUT_OF_RANGE | 400 | Bad Request |
| 12 | UNIMPLEMENTED | 501 | Not Implemented |
| 13 | INTERNAL | 500 | Internal Server Error |
| 14 | UNAVAILABLE | 503 | Service Unavailable |
| 15 | DATA_LOSS | 500 | Internal Server Error |
| 16 | UNAUTHENTICATED | 401 | Unauthorized |

#### 1.验证错误（标准 Go 语言错误）
> 根据错误映射，将会返回 500。

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return nil, errors.New("error triggered manually")
}
```

```
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
{
    "error":{
        "code":500,
        "status":"Internal Server Error",
        "message":"error triggered manually",
        "details":[]
    }
}
```

#### 2.验证错误（grpc 错误）
我们需要通过 status.New() 来创建 GRPC 错误。我们推荐使用 rkerror 库中的函数来创建错误。

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return nil, rkerror.PermissionDenied("permission denied manually").Err()
}
```

```
curl "localhost:8080/v1/greeter?name=rk-dev"
{
    "error":{
        "code":403,
        "status":"Forbidden",
        "message":"permission denied manually",
        "details":[
            {
                "code":7,
                "status":"PermissionDenied",
                "message":"[from-grpc] permission denied manually"
            }
        ]
    }
}
```