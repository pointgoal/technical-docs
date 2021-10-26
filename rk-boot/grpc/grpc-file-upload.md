# GRPC: 如何实现文件上传 Restful API ？

## 介绍

本文将介绍如何在 gRPC 微服务中实现文件上传 Restful API。

> 为什么需要这么一篇文章？
>
> gRPC 里我们可以通过 Streaming 来互传大文件，不过通过 grpc-gateway on gRPC 我们是无法实现的。
> 因此，需要绕过 gRPC 直接在 grpc-gateway 中添加 API。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 gRPC 服务。

> 请访问如下地址获取完整教程：
> - https://rkdev.info/cn
> - https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
[rk-boot](https://github.com/rookie-ninja/rk-boot) 默认集成了 grpc-gateway，并且会默认启动。

### 1.创建 boot.yaml
```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
```

### 2.创建 main.go 
注意，grpcEntry.GwMux.HandlePath() 一定要写到 boot.Bootstrap() 之后，否则会出现 Panic。

```
package main

import (
	"context"
	"fmt"
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

	// Attachment upload from http/s handled manually
	grpcEntry.GwMux.HandlePath("POST", "/v1/files", handleBinaryFileUpload)

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

func handleBinaryFileUpload(w http.ResponseWriter, req *http.Request, params map[string]string) {
	err := req.ParseForm()
	if err != nil {
		http.Error(w, fmt.Sprintf("failed to parse form: %s", err.Error()), http.StatusBadRequest)
		return
	}

	f, header, err := req.FormFile("attachment")
	if err != nil {
		http.Error(w, fmt.Sprintf("failed to get file 'attachment': %s", err.Error()), http.StatusBadRequest)
		return
	}
	defer f.Close()

	fmt.Println(header)

	//
	// Now do something with the io.Reader in `f`, i.e. read it into a buffer or stream it to a gRPC client side stream.
	// Also `header` will contain the filename, size etc of the original file.
	//
}
```

### 3.验证
```
$ curl -X POST -F "attachment=@xxx.txt" localhost:8080/v1/files
```