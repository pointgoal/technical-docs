# gRPC: How to shut down service gracefully?

## Introduction
Provide graceful shutdown logic in gRPC micro-service easily with [rk-boot](https://github.com/rookie-ninja/rk-boot).

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
```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
```

### II. Create main.go 
Add shutdownhook functions via AddShutdownHookFunc().


```
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-gin/interceptor/context"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()
    
    // Add shutdown hook function
	boot.AddShutdownHookFunc("shutdown-hook", func() {
		fmt.Println("shutting down")
	})

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### III. Start main.go
```
$ go run main.go
```

### IV. ctrl-c
Shutting down service via ctrl-c.

```
shutting down
```