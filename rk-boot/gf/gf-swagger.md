# GoFrame 框架: 添加 Swagger UI

## 介绍
本文将介绍如何在 [GoFrame](https://github.com/gogf/gf) 框架之上提供 Swagger UI。

> 请访问如下地址获取完整 GoFrame 教程：
>
- https://rkdocs.netlify.app/cn

## 先决条件
[GoFrame](https://github.com/gogf/gf) 没有自带生成 Swagger UI 配置文件的功能。

我们需要安装 [swag](https://github.com/swaggo/swag) 命令行工具来生成 Swagger UI 配置文件。

> 安装选项 1：通过 [RK 命令行](https://github.com/rookie-ninja/rk)

```
# Install RK CMD
$ go get -u github.com/rookie-ninja/rk/cmd/rk

# Install swag with rk
$ rk install swag
```

> 安装选项 2：通过 [swag](https://github.com/swaggo/swag) 官网

```
$ go get -u github.com/swaggo/swag/cmd/swag
```

## 安装 rk-boot
我们介绍 [rk-boot](https://github.com/rookie-ninja/rk-boot) 库，用户可以快速搭建基于 GoFrame 框架的微服务。

- [文档](https://rkdocs.netlify.app/cn/)
- [源代码](https://github.com/rookie-ninja/rk-boot)
- [例子](https://rkdocs.netlify.app/cn/docs/bootstrapper/user-guide/gf-golang/basic/swagger-ui/)

```go
go get github.com/rookie-ninja/rk-boot
go get github.com/rookie-ninja/rk-gf
```

## 快速开始
### 1.创建 boot.yaml
boot.yaml 文件会告诉 [rk-boot](https://github.com/rookie-ninja/rk-boot) 如何启动 GoFrame 服务，下面的例子中，我们指定了端口，Swagger UI 的 json 文件路径。

```
---
gf:
  - name: greeter        # Required
    port: 8080           # Required
    enabled: true        # Required
    sw:
      enabled: true      # Optional, default: false
      jsonPath: "docs"   # Optional, default: ""
#      path: "sw"        # Default value is "sw", change it as needed
#      headers: []       # Headers that will be set while accessing swagger UI main page.
```

### 2.创建 main.go 
为了能让 [swag](https://github.com/swaggo/swag) 命令行生成 Swagger UI 参数文件，我们需要在代码中写注释。

详情可参考 [swag](https://github.com/swaggo/swag) 官方文档。

```
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"fmt"
	"github.com/gogf/gf/v2/net/ghttp"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-gf/boot"
	"net/http"
)

// @title RK Swagger for GoFrame
// @version 1.0
// @description This is a greeter service with rk-boot.

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Register handler
	gfEntry := boot.GetEntry("greeter").(*rkgf.GfEntry)
	gfEntry.Server.BindHandler("/v1/greeter", Greeter)

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}

// @Summary Greeter service
// @Id 1
// @version 1.0
// @produce application/json
// @Param name query string true "Input name"
// @Success 200 {object} GreeterResponse
// @Router /v1/greeter [get]
func Greeter(ctx *ghttp.Request) {
	ctx.Response.WriteHeader(http.StatusOK)
	err := ctx.Response.WriteJson(&GreeterResponse{
		Message: fmt.Sprintf("Hello %s!", ctx.GetQuery("name")),
	})

	if err != nil {
		panic(err)
	}
}

// Response.
type GreeterResponse struct {
	Message string
}

```

### 3.生成 swagger 参数文件
默认会在 docs 文件夹里面创建三个文件。rk-boot 会使用 swagger.json 来初始化 Swagger UI 界面。

```
$ swag init

$ tree
.
├── boot.yaml
├── docs
│   ├── docs.go
│   ├── swagger.json
│   └── swagger.yaml
├── go.mod
├── go.sum
└── main.go

1 directory, 7 files
```

### 4.验证 
访问：[localhost:8080/sw](http://localhost:8080/sw)

![image](img/gf-sw-api.png)