# GRPC: 调整 gRPC 数据传输大小限制

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 调整 gRPC 数据传输大小限制。

之前版本的 grpc-go，客户端端也需要调整传输大小，新版本的 grpc-go 已经不需要了。
例子里使用的是 google.golang.org/grpc v1.38.0 版本。

> **什么是 gRPC 数据传输大小限制？** 
>
> gRPC 服务端默认最大数据传输大小为 4MB，有些时候，我们需要传输更大的数据，比如大图片。

**请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
[rk-boot](https://github.com/rookie-ninja/rk-boot) 支持通过代码 & YAML 文件的方式调整大小限制。

为了完整演示，我们创建一个 greeter API。

### 1.创建 protobuf 相关文件
我们使用 buf 命令行来编译 protobuf，需要创建如下几个文件。

| 文件名 | 描述 |
| ---- | ---- |
| api/v1/greeter.proto | protobuf 文件 |
| buf.yaml | 告诉 buf 命令行在哪里寻找 protobuf 文件 |
| buf.gen.yaml | 告诉 buf 命令行如何编译 protobuf 文件 |

- **api/v1/greeter.proto**

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "api/v1/greeter";

service Greeter {
  rpc Greeter (GreeterRequest) returns (GreeterResponse) {}
}

message GreeterRequest {
  bytes msg = 1;
}

message GreeterResponse {}
```

- **buf.yaml**

```yaml
version: v1beta1
name: github.com/rk-dev/rk-demo
build:
  roots:
    - api
```

- **buf.gen.yaml**

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
```

- **编译 protobuf 文件**

```shell
$ buf generate
```

### 2.创建 boot.yaml
我们通过 boot.yaml 方式来【取消】大小限制，通过 boot.yaml 我们可以取消限制，但是无法调整限制。

调整限制的话，可以通过代码调整，我们也会在下面介绍。

```yaml
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    noRecvMsgSizeLimit: true
```

### 4.创建 server.go 
我们实现了 greeter 接口。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-demo/api/gen/v1"
	"google.golang.org/grpc"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Get grpc entry with name
	grpcEntry := boot.GetGrpcEntry("greeter")
	grpcEntry.AddRegFuncGrpc(registerGreeter)

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

func registerGreeter(server *grpc.Server) {
	greeter.RegisterGreeterServer(server, &GreeterServer{})
}

type GreeterServer struct{}

func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return &greeter.GreeterResponse{}, nil
}
```

### 5.创建 client.go
我们试着传输 10MB 的数据。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/rookie-ninja/rk-demo/api/gen/v1"
	"google.golang.org/grpc"
	"log"
)

func main() {
	opts := []grpc.DialOption{
		grpc.WithInsecure(),
		grpc.WithBlock(),
	}

	// 1: Create grpc client
	conn, client := createGreeterClient(opts...)
	defer conn.Close()

	kb := 1024
	mb := 1024*kb

	// 2: Call server with 10mb size of data
	if _, err := client.Greeter(context.Background(), &greeter.GreeterRequest{Msg: make([]byte, 10*mb, 10*mb)}); err != nil {
		panic(err)
	}
}

func createGreeterClient(opts ...grpc.DialOption) (*grpc.ClientConn, greeter.GreeterClient) {
	// 1: Set up a connection to the server.
	conn, err := grpc.DialContext(context.Background(), "localhost:8080", opts...)
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}

	// 2: Create grpc client
	client := greeter.NewGreeterClient(conn)

	return conn, client
}
```

### 6.文件夹结构

```
.
├── api
│   ├── gen
│   │   └── v1
│   │       ├── greeter.pb.go
│   │       └── greeter_grpc.pb.go
│   └── v1
│       └── greeter.proto
├── boot.yaml
├── buf.gen.yaml
├── buf.yaml
├── client.go
├── go.mod
├── go.sum
└── server.go
```

### 7.验证
不会出现任何错误。

```
$ go run server.go
$ go run client.go
```

因为服务端默认只允许接收 4MB 大小的数据，如果我们在 boot.yaml 里把 noRecvMsgSizeLimit 设置成 false，会得到如下错误。

```
rpc error: code = ResourceExhausted desc = grpc: received message larger than max (10485765 vs. 4194304)
```

## 调整数据传输大小
上次的例子中，我们使用 noRecvMsgSizeLimit 选项取消了 gRPC 服务端的大小限制，这次，我们试着调整大小。
还是使用上面的 protobuf 文件。

### 1.修改 boot.yaml
这次我们把 noRecvMsgSizeLimit 设置成 false。

```yaml
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    noRecvMsgSizeLimit: false
```

### 2.修改 server.go
我们通过 AddServerOptions() 函数设置服务端接收最大值。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-demo/api/gen/v1"
	"google.golang.org/grpc"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Get grpc entry with name
	grpcEntry := boot.GetGrpcEntry("greeter")
	grpcEntry.AddRegFuncGrpc(registerGreeter)
	
	// *************************************** //
	// *** Set server receive size to 20MB ***
	// *************************************** //
	kb := 1024
	mb := 1024*kb
	grpcEntry.AddServerOptions(grpc.MaxRecvMsgSize(20*mb))

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

func registerGreeter(server *grpc.Server) {
	greeter.RegisterGreeterServer(server, &GreeterServer{})
}

type GreeterServer struct{}

func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return &greeter.GreeterResponse{}, nil
}
```

### 3.验证
不会出现任何错误。

```
$ go run server.go
$ go run client.go
```

如果我们发送的数据大于 20mb，会出现如下错误。

```
rpc error: code = ResourceExhausted desc = grpc: received message larger than max (31457285 vs. 20971520)
```
