# GRPC: 如何优雅关闭进程（graceful shutdown）？

## 介绍

本文将介绍优雅关闭 gRPC 微服务。

> 什么是优雅关闭？
>
> 在进程收到关闭信号时，我们需要关闭后台运行的逻辑，比如，MySQL 连接等等。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 gRPC 服务。

> 请访问如下地址获取完整教程：
>
> - https://rkdev.info/cn
>
> - https://rkdocs.netlify.app/cn (备用)

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
### 1.创建 boot.yaml
```
---
grpc:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
```

### 2.创建 main.go
通过 AddShutdownHookFunc() 来添加 shutdownhook 函数。

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

### 3.启动 main.go
```
$ go run main.go
```

### 4.ctrl-c
通过 ctrl-c 关闭程序，我们会看到打印如下信息。

```
shutting down
```