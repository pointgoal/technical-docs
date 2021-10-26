# GRPC: 使用 Buf 快速编译 GRPC proto 文件

## 介绍
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

> 请访问如下地址获取完整教程：
> - https://rkdev.info/cn
> - https://rkdocs.netlify.app/cn (备用)

## 使用 [Buf](https://docs.buf.build) 快速编译
我们可以通过 [Buf](https://docs.buf.build) 快速配置编译流程。虽然前期需要一定的配置，但是比起写复杂的脚本，要简单安全的多。

下面我们就通过一个例子来浏览一下。

## 例子
我们以简单的 Hello World 为例子，一步一步生成基于 GRPC 的微服务。
- [例子](https://github.com/rookie-ninja/rk-demo/tree/master/grpc/getting-started)
- [文档](https://rkdev.info/cn/docs/bootstrapper/getting-started/grpc-golang/)

### 第一步：安装命令行
> 建议通过 [rk](https://github.com/rookie-ninja/rk) 命令行，快速安装所需要的工具。
```shell script
# Install RK CLI
$ go get -u github.com/rookie-ninja/rk/cmd/rk

# List available installation
$ rk install
COMMANDS:
    buf                      install buf on local machine
    cfssl                    install cfssl on local machine
    cfssljson                install cfssljson on local machine
    gocov                    install gocov on local machine
    golangci-lint            install golangci-lint on local machine
    mockgen                  install mockgen on local machine
    pkger                    install pkger on local machine
    protobuf                 install protobuf on local machine
    protoc-gen-doc           install protoc-gen-doc on local machine
    protoc-gen-go            install protoc-gen-go on local machine
    protoc-gen-go-grpc       install protoc-gen-go-grpc on local machne
    protoc-gen-grpc-gateway  install protoc-gen-grpc-gateway on local machine
    protoc-gen-openapiv2     install protoc-gen-openapiv2 on local machine
    swag                     install swag on local machine
    rk-std                   install rk standard environment on local machine
    help, h                  Shows a list of commands or help for one command

# Install protobuf, buf, protoc-gen-go, protoc-gen-go-grpc, protoc-gen-grpc-gateway, protoc-gen-openapiv2
$ rk install protobuf
$ rk install protoc-gen-go
$ rk install protoc-gen-go-grpc
$ rk install protoc-gen-go-grpc-gateway
$ rk install protoc-gen-openapiv2
$ rk install buf
```

> 也可以自行在官网安装

| 工具 | 安装 |
| ---- | ---- |
| [protobuf](https://github.com/protocolbuffers/protobuf) | [Install](http://google.github.io/proto-lens/installing-protoc.html) |
| [protoc-gen-go](https://github.com/golang/protobuf/tree/master/protoc-gen-go) | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-go-grpc](https://github.com/grpc/grpc-go) | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |
| [protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway) | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |

### 第二步：创建 api/v1/greeter.proto
```protobuf
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

### 第三步：创建 api/v1/gw_mapping.yaml
我们通过 gw_mapping.yaml 文件来映射 GRPC -> Restful API。这样我们可以避免在 *.proto 文件里写一堆 option 代码。

具体语法，可以参考：https://github.com/googleapis/googleapis/blob/master/google/api/http.proto

```yaml
type: google.api.Service
config_version: 3

# Please refer google.api.Http in https://github.com/googleapis/googleapis/blob/master/google/api/http.proto file for details.
http:
  rules:
    - selector: api.v1.Greeter.Greeter
      get: /api/v1/greeter
```

### 第四步：创建 buf.yaml
我们通过 buf.yaml 告诉 buf 在那里寻找 proto 文件。我们指定 api/ 文件夹。

```yaml
version: v1beta1
name: github.com/rk-dev/rk-demo
build:
  roots:
    - api
```

### 第五步：创建 buf.gen.yaml
下面的参数，告诉 buf 编译的时候，应该做哪些事情。我们的文件中，主要做了如下的事情。
- 指定编译后的文件，放到 api/gen 文件夹中
- 编译 proto 文件
- 编译 GRPC 相关的 proto 文件
- 编译 GRPC-Gateway 相关的 proto 文件
- 从 proto 文件，编译出 openapi-v2 相关的文件（Swagger）

```yaml
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

### 第六步：编译 proto
上述配置都完成以后，无论 *.proto 文件如何修改，我们只要运行 buf generate，即可编译 *.proto 文件，非常方便。

```shell script
$ buf generate
```

> 如下的文件会被创建。

```shell script
$ tree api/gen 
api/gen
└── v1
    ├── greeter.pb.go
    ├── greeter.pb.gw.go
    ├── greeter.swagger.json
    └── greeter_grpc.pb.go
 
1 directory, 4 files
```

### 第七步：在 main.go 中引用
我们已经编译好了 *.proto 文件，剩下的就是在 main.go 文件中引用了刚刚生成的 proto API 了。

这里，我们介绍 [rk-boot](https://github.com/rookie-ninja/rk-boot) 库，通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 用户可以快速搭建基于 GRPC 的微服务。
- [文档](https://rkdev.info/cn/docs/bootstrapper/getting-started/grpc-golang/)
- [源代码](https://github.com/rookie-ninja/rk-boot)
- [例子](https://github.com/rookie-ninja/rk-demo/tree/master/grpc/getting-started)

#### 创建 boot.yaml
```yaml
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    sw:
      enabled: true               # Enable Swagger UI
      jsonPath: "api/gen/v1"      # Boot will look for swagger config files from this folder
```

#### 创建 main.go
```go
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

#### 文件夹结构
```shell script
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

#### 验证
```shell script
$ go run main.go
```

```shell script
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
{"message":"Hello rk-dev!"}
```
