# gRPC: gRPC 接口与 Restful API 混合使用

## 介绍

本文将介绍如何在 gRPC 微服务中混合使用 Restful API。

这里我们并不是把 gRPC 接口转换成 Restful API，而是让不同的 gRPC 接口与 Restful API 共存。
[grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/docs/operations/inject_router/) 已经支持了此功能。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 gRPC 服务。

> **请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
这个例子中，不会编写任何 gRPC 接口，我们会在 gRPC 服务中加入一个独立的 Restful API。

### 1.创建 boot.yaml

```yaml
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
```

### 2.创建 main.go
在 grpc-gateway 中创建一个 GET /custom 方法。

通过 **boot.GetGrpcEntry("greeter").GwMux.HandlePath()** 方法来加入自定义的接口。比如文件上传。

```go
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
	"net/http"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Get grpc entry with name
	grpcEntry := boot.GetGrpcEntry("greeter")
    
    // !!!!!!
    // This codes should be located after Bootstrap()
	grpcEntry.GwMux.HandlePath("GET", "/custom", func(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
		w.Write([]byte("Custom routes!"))
	})

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.验证

```shell
$ curl "localhost:8080/custom"
Custom routes!
```