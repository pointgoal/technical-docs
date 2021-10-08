# GRPC: How to make gRPC support Restful API?

## Introduction
In order to make gRPC microservice support restful API, we need to use [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)。

## Use rk-boot
We introduce [rk-boot](https://github.com/rookie-ninja/rk-boot) which is a library can be used to create golang microservice with grpc in a convenient way.
- [Docs](https://rkdev.info/docs/bootstrapper/getting-started/grpc-golang/)
- [Source code](https://github.com/rookie-ninja/rk-boot)
- [Example](https://github.com/rookie-ninja/rk-demo/tree/master/grpc/getting-started)

### Requisition
In order to compile *.proto files to *.go files, we need couple of command line tools.

For example, in golang, we need at least 4 different command line tools for gRpc microservice.

| Tool | Description | Installation |
| ---- | ---- | ---- |
| [protobuf](https://github.com/protocolbuffers/protobuf) | protocol buffer | [Install](http://google.github.io/proto-lens/installing-protoc.html) |
| [protoc-gen-go](https://github.com/golang/protobuf/tree/master/protoc-gen-go) | Plugin for the Google protocol buffer compiler to generate Go code.. | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-go-grpc](https://github.com/grpc/grpc-go) | This project aims to provide that HTTP+JSON interface to your gRPC service. | [Install](https://grpc.io/docs/languages/go/quickstart/) |
| [protoc-gen-grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) | plugin for Google protocol buffer compiler to generate a reverse-proxy, which converts incoming RESTful HTTP/1 requests gRPC invocation. | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |
| [protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway) | plugin for Google protocol buffer compiler to generate open API config file. | [Install](https://github.com/grpc-ecosystem/grpc-gateway#installation) |

> Please refer to my previous article: GRPC: How to compile proto files with buf quickly and safely.
> or visit: https://rkdev.info/docs/bootstrapper/user-guide/grpc-golang/basic/grpc-gateway/

### Quick start
#### 1.Create api/v1/greeter.proto
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

#### 2.Create api/v1/gw_mapping.yaml
```
type: google.api.Service
config_version: 3

# Please refer google.api.Http in https://github.com/googleapis/googleapis/blob/master/google/api/http.proto file for details.
http:
  rules:
    - selector: api.v1.Greeter.Greeter
      get: /api/v1/greeter
```

#### 3.Create buf.yaml
```
version: v1beta1
name: github.com/rk-dev/rk-demo
build:
  roots:
    - api
```

#### 4.Create buf.gen.yaml
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

#### 5.Compile proto file
```
$ buf generate
```

> There will be bellow files generated.
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

#### 6.Create boot.yaml

```
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
    enableReflection: true
```

#### 7. Create main.go
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

#### 8.Full structure
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

#### 9.Validate
```
$ go run main.go
```

- Validate Restful API

```
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
{"message":"Hello rk-dev!"}
```

- Validate gRPC

```
$ grpcurl -d '{"name": "rk-dev"}' -plaintext localhost:8080 api.v1.Greeter.Greeter
{
  "message": "Hello rk-dev!"
}
```

### Gateway server option
> RK introduced gateway server options with bellow functionality.
>
> [rkgrpc.RkGwServerMuxOptions](https://github.com/rookie-ninja/rk-grpc/blob/master/boot/gw_server_options.go)

| Functionality | Description |
| ---- | ---- |
| HttpErrorHandler | Mainly copied from original gateway error handler. RK wrap errors to RK style error response. |
| MarshalerOption | protojson.MarshalOptions{UseProtoNames: false, EmitUnpopulated: true},UnmarshalOptions: protojson.UnmarshalOptions{}} |
| Metadata | Inject x-forwarded-method, x-forwarded-path, x-forwarded-scheme, x-forwarded-user-agent and x-forwarded-remote-addr into grpc metadata |
| OutgoingHeaderMatcher | Bypass all grpc metadata to http header without prefix |
| IncomingHeaderMatcher | Bypass all http header to grpc metadata without prefix |

### 1.Enable enableRkGwOption
We add logging interceptor for validation.

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

#### 2.Validate log
```
$ curl "localhost:8080/api/v1/greeter?name=rk-dev"
```

> gwMethod, gwPath, gwScheme, gwUserAgent would be logged
 
```
------------------------------------------------------------------------
endTime=2021-07-09T21:03:43.518106+08:00
...
payloads={"grpcMethod":"Greeter","grpcService":"api.v1.Greeter","grpcType":"unaryServer","gwMethod":"GET","gwPath":"/v1/greeter","gwScheme":"http","gwUserAgent":"curl/7.64.1"}
...
```

#### 3.Validate incoming metadata
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

### Error mapping
It is important to understand grpc-gateway error to grpc error mapping.

Here is the default error mapping defined in grpc-gateway.

| GRPC Error | GRPC Error Str | Gateway(Http) Error Code | Gateway(Http) Error Str |
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

#### 1.Validate error(Standard go error)
> Based on default error mapping, we expect 500 response code.

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

#### 2.Validate error(grpc error)
> grpc needs to specify errors with status.New().
> 
> We recommend use rkerror to generate errors as bellow.

```
func (server *GreeterServer) Greeter(ctx context.Context, request *greeter.GreeterRequest) (*greeter.GreeterResponse, error) {
	return nil, rkerror.PermissionDenied("permission denied manually").Err()
}
```

```
curl "localhost:8080/api/v1/greeter?name=rk-dev"
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
