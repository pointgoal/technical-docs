# gRPC: Open multiple ports with rk-boot

## Introduction
Open multiple ports in a server easily with [rk-boot](https://github.com/rookie-ninja/rk-boot).

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

### I. Create boot.yaml
We enabled commonService which contains a couple commonly used API like /rk/v1/healthy.

**By default, rk-boot will enable grpc-gateway automatically.**

```
---
grpc:
  - name: alice
    port: 1949
    enabled: true
    commonService:
      enabled: true
  - name: bob
    port: 2008
    enabled: true
    commonService:
      enabled: true
```

### II. Create main.go 
Register gRPC APIs via **boot.GetGrpcEntry("xxx")**.

```
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
)

func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Get alice Entry via: 
	//   boot.GetGrpcEntry("alice")
	// 
	// Get bob Entry via: 
	//   boot.GetGrpcEntry("bob")

	// Bootstrap
	boot.Bootstrap(context.Background())
	
	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### III. Directory hierarchy
```
$ tree
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

### IV. Start main.go

```
$ go run main.go
```

### V. Validate
- Send request to **1949**

```
$ curl -X GET localhost:1949/rk/v1/healthy
{"healthy":true}
```

- Send request to **2008**
```
$ curl -X GET localhost:2008/rk/v1/healthy
{"healthy":true}
```

## Start 1949 only
**--rkset** is the flag to override values in boot.yaml file.

Since grpc is array in boot.yaml, we can access values with **grpc[x]**.

### I. Start main.go with --rkset
```
$ go build main.go
$ ./main --rkset "grpc[1].enabled=false"
```

### II. Validate
- Send request to **1949**

```
$ curl -X GET localhost:1949/rk/v1/healthy
{"healthy":true}
```

- Send request to **2008**
```
$ curl -X GET localhost:2008/rk/v1/healthy
curl: (7) Failed to connect to localhost port 2008: Connection refused
```

## Override boot.yaml with rkset
**--rkset** is the flag to override values in boot.yaml file.

Use comma to separate multiple overrides.

| Type | Example |
| ---- | ---- |
| Map | app.description="This is description" |
| List | grpc[0].name="alice",grpc[0].port=8081 |
