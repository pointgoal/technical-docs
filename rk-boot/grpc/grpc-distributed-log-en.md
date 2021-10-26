# gRPC: How to trace logs in distributed system?

## Introduction
Support tracing of API logs in distributed gRPC micro-service easily with [rk-boot](https://github.com/rookie-ninja/rk-boot) and [rk-grpc](https://github.com/rookie-ninja/rk-grpc).

In order to compile *.proto files to *.go files, we need couple of command line tools.

For example, in golang, we need at least 4 different command line tools for gRpc microservice.

| Tool | Description | Installation |
| ---- | ---- | ---- |
| [protobuf](https://github.com/protocolbuffers/protobuf) | protocol buffer | [Install](http://google.github.io/proto-lens/installing-protoc.html) |
| [protoc-gen-go](https://github.com/golang/protobuf/tree/master/protoc-gen-go) | Plugin for the Google protocol buffer compiler to generate Go code.. | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-go-grpc](https://github.com/grpc/grpc-go) | This project aims to provide that HTTP+JSON interface to your gRPC service. | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) | plugin for Google protocol buffer compiler to generate a reverse-proxy, which converts incoming RESTful HTTP/1 requests gRPC invocation. | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |
| [protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway) | plugin for Google protocol buffer compiler to generate open API config file. | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |

## Use [Buf](https://docs.buf.build)
[Buf](https://docs.buf.build) is also a command line tools integrate above tools into a single command. It is much easier and safer to use while compiling complex protocol buffer files.

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

[rk-boot](https://github.com/rookie-ninja/rk-boot) integrate grpc-gateway and will start grpc-gateway automatically with same port of gRPC.

![image](img/trace-arch.png)

### I. Create api/v1/greeter.proto 
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

### II. Create api/v1/gw_mapping.yaml 
```
type: google.api.Service
config_version: 3

# Please refer google.api.Http in https://github.com/googleapis/googleapis/blob/master/google/api/http.proto file for details.
http:
  rules:
    - selector: api.v1.Greeter.Greeter
      get: /api/v1/greeter
```

### III. Create buf.yaml
```
version: v1beta1
name: github.com/rk-dev/rk-demo
build:
  roots:
    - api
```

### IV. Create buf.gen.yaml
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

### V. Compile proto files
```
$ buf generate
```

> Directory hierarchy looks like bellow.

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

### VI. Create bootA.yaml & serverA.go
Server-A will listen to port of 1949 and send request to Server-B.

Server-A will inject tracing info to context via rkgrpcctx.InjectSpanToNewContext().

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 1949                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    interceptors:
      loggingZap:
        enabled: true
      meta:
        enabled: true
      tracingTelemetry:
        enabled: true
```

```
package main

import (
	"context"
	"demo/api/gen/v1"
	"fmt"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-grpc/interceptor/context"
	"google.golang.org/grpc"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot(rkboot.WithBootConfigPath("bootA.yaml"))

	// Get grpc entry with name
	grpcEntry := boot.GetGrpcEntry("greeter")
	grpcEntry.AddRegFuncGrpc(registerGreeter)
	grpcEntry.AddRegFuncGw(greeter.RegisterGreeterHandlerFromEndpoint)

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
	// Call serverB at 2008 with grpc client
	opts := []grpc.DialOption{
		grpc.WithBlock(),
		grpc.WithInsecure(),
	}
	conn, _ := grpc.Dial("localhost:2008", opts...)
	defer conn.Close()
	client := greeter.NewGreeterClient(conn)

	// Inject current trace information into context
	newCtx := rkgrpcctx.InjectSpanToNewContext(ctx)
	client.Greeter(newCtx, &greeter.GreeterRequest{Name: "A"})

	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

### VII. Create bootB.yaml & serverB.go
Server-B will listen port of 2008.

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 2008                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    interceptors:
      loggingZap:
        enabled: true
      meta:
        enabled: true
      tracingTelemetry:
        enabled: true
```

```
package main

import (
	"context"
	"demo/api/gen/v1"
	"fmt"
	"github.com/rookie-ninja/rk-boot"
	"google.golang.org/grpc"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot(rkboot.WithBootConfigPath("bootB.yaml"))

	// Get grpc entry with name
	grpcEntry := boot.GetGrpcEntry("greeter")
	grpcEntry.AddRegFuncGrpc(registerGreeterB)
	grpcEntry.AddRegFuncGw(greeter.RegisterGreeterHandlerFromEndpoint)

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

func registerGreeterB(server *grpc.Server) {
	greeter.RegisterGreeterServer(server, &GreeterServerB{})
}

type GreeterServerB struct{}

func (server *GreeterServerB) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return &greeter.GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", request.Name),
	}, nil
}
```

### VIII. Directory Hierarchy

```
├── api
│   ├── gen
│   │   └── v1
│   │     ├── greeter.pb.go
│   │     ├── greeter.pb.gw.go
│   │     ├── greeter.swagger.json
│   │     └── greeter_grpc.pb.go
│   └── v1
│       ├── greeter.proto
│       └── gw_mapping.yaml
├── bootA.yaml
├── bootB.yaml
├── buf.gen.yaml
├── buf.yaml
├── go.mod
├── go.sum
├── serverA.go
└── serverB.go
```

### IX. Start ServerA & ServerB
```
$ go run serverA.go
$ go run serverB.go
```

### X. Send request to ServerA
```
$ curl "localhost:1949/api/v1/greeter?name=rk-dev"
```

### XI. Validate logs
Two servers will have same traceId in event log but different requestId and eventId.

- ServerA

```
------------------------------------------------------------------------
endTime=2021-10-20T00:02:21.739688+08:00
...
ids={"eventId":"0d145356-998a-4999-ab62-6f1b805274a0","requestId":"0d145356-998a-4999-ab62-6f1b805274a0","traceId":"c36a45eb076066df39fa407174012369"}
...
operation=/api.v1.Greeter/Greeter
resCode=OK
eventStatus=Ended
EOE
```

- ServerB
```
------------------------------------------------------------------------
endTime=2021-10-20T00:02:21.739125+08:00
...
ids={"eventId":"8858a6eb-e953-42ad-bdc3-c466bbbd798e","requestId":"8858a6eb-e953-42ad-bdc3-c466bbbd798e","traceId":"c36a45eb076066df39fa407174012369"}
...
operation=/api.v1.Greeter/Greeter
resCode=OK
eventStatus=Ended
EOE
```

## Concept
### EventId
EventId would be generated if logging interceptor/middleware was enabled.

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 1949                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
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
Generated by meta interceptor/middleware automatically if user enabled bellow:

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 1949                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
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

> If meta interceptor/middleware was enabled or requestId was enabled by user, then eventId will be override by requestId.
> 
> Simply, eventId will always the same as requestId

```
rkgrpcctx.AddHeaderToClient(ctx, rkgrpcctx.RequestIdKey, "overridden-request-id")
```

```
------------------------------------------------------------------------
...
ids={"eventId":"overridden-request-id","requestId":"overridden-request-id"}
...
```

### TraceId
Generated if user enable tracing interceptor/middleware by user as bellow:

```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 1949                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
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

```
{
    "eventId":"dd19cf9a-c7be-486c-b29d-7af777a78ebe",
    "requestId":"dd19cf9a-c7be-486c-b29d-7af777a78ebe",
    "traceId":"316a7b475ff500a76bfcd6147036951c"
}
```
