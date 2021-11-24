# Echo 框架：优雅关闭进程

## 介绍
通过一个完整例子，介绍如何优雅关闭 Echo 微服务。

> **什么是优雅关闭？**
>
> 在进程收到关闭信号时，我们需要关闭后台运行的逻辑，比如，MySQL 连接等等。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Echo 框架微服务。

请访问如下地址获取完整教程：

- https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
### 1.创建 boot.yaml
boot.yaml 文件会告诉 [rk-boot](https://github.com/rookie-ninja/rk-boot) 如何启动 Echo 服务。

```yaml
---
echo:
  - name: greeter                   # Name of grpc entry
    port: 8080                      # Port of grpc entry
    enabled: true                   # Enable grpc entry
```

### 2.创建 main.go
通过 AddShutdownHookFunc() 来添加 shutdownhook 函数。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"fmt"
	"github.com/rookie-ninja/rk-boot"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

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

