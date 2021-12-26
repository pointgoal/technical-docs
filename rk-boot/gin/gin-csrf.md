# Gin 安全篇-3: 快速实现 CSRF 验证

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 实现服务端 CSRF 验证逻辑。

> **什么是 CSRF？**
>
> 跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。
>
> 跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

![image](https://bkimg.cdn.bcebos.com/pic/86d6277f9e2f0708efe75721e024b899a901f235?x-bce-process=image/watermark,image_d2F0ZXIvYmFpa2U4MA==,g_7,xp_5,yp_5/format,f_auto)

> **有什么防御方法？**
>
> 流行的防御方法有如下几种，我们通过例子实现【添加校验 Token】的防御。
>
> 1: 令牌同步模式
> 2：检查 Referer 字段
> 3：添加校验 Token

**请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn

## 安装
```go 
go get github.com/rookie-ninja/rk-boot/gin
```

## 快速开始
### 1.创建 boot.yaml
boot.yaml 文件会告诉 rk-boot 如何启动 Gin 服务。

在下面的 YAML 文件中，我们声明了一件事：
- 开启 CSRF 拦截器，使用默认参数。拦截器会检查请求 Header 里 X-CSRF-Token 的值，判断 Token 是否正确。

```yaml
---
gin:
  - name: greeter                     # Required
    port: 8080                        # Required
    enabled: true                     # Required
    interceptors:
      csrf:
        enabled: true                 # Optional, default: false
```

### 2.创建 main.go
我们在 Gin 里添加两个 Restful API。

- GET /v1/greeter: 返回服务端生成的 CSRF Token
- POST /v1/greeter: 验证 CSRF Token

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package main

import (
	"context"
	"github.com/gin-gonic/gin"
	"github.com/rookie-ninja/rk-boot"
	"github.com/rookie-ninja/rk-boot/gin"
	"net/http"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	ginEntry := rkbootgin.GetGinEntry("greeter")
	// Register /v1/greeter GET
	ginEntry.Router.GET("/v1/greeter", func(ctx *gin.Context) {
		ctx.JSON(http.StatusOK, "Hello!")
	})
	// Register /v1/greeter POST
	ginEntry.Router.POST("/v1/greeter", func(ctx *gin.Context) {
		ctx.JSON(http.StatusOK, "Hello!")
	})

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.文件夹结构
```
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

- go.mod

```
module github.com/rookie-ninja/rk-demo

go 1.16

require (
	github.com/rookie-ninja/rk-boot v1.4.0
	github.com/rookie-ninja/rk-boot/gin v1.2.12
)
```

### 4.验证
- 发送 GET 请求到 /v1/greeter，我们将获得 CSRF Token。

```
$ curl -X GET -vs localhost:8080/v1/greeter
...
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< Set-Cookie: _csrf=XVlBzgbaiCMRAjWwhTHctcuAxhxKQFDa; Expires=Mon, 27 Dec 2021 09:35:20 GMT
< Vary: Cookie
< Date: Sun, 26 Dec 2021 09:35:20 GMT
< Content-Length: 8
< 
* Connection #0 to host localhost left intact
"Hello!"* 
```

- 发送 POST 请求到 /v1/greeter，提供合法 CSRF Token。

```
$ curl -X POST -v --cookie "_csrf=my-test-csrf-token" -H "X-CSRF-Token:my-test-csrf-token" localhost:8080/v1/greeter
...
> Cookie: _csrf=my-test-csrf-token
> X-CSRF-Token:my-test-csrf-token
> 
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< Set-Cookie: _csrf=my-test-csrf-token; Expires=Mon, 27 Dec 2021 09:35:43 GMT
< Vary: Cookie
< Date: Sun, 26 Dec 2021 09:35:43 GMT
< Content-Length: 8
< 
* Connection #0 to host localhost left intact
"Hello!"* 
```

- 发送 POST 请求到 /v1/greeter，提供非法 CSRF Token。

```
$ curl -X POST -v -H "X-CSRF-Token:my-test-csrf-token" localhost:8080/v1/greeter
...
> X-CSRF-Token:my-test-csrf-token
> 
< HTTP/1.1 403 Forbidden
< Content-Type: application/json; charset=utf-8
< Date: Sun, 26 Dec 2021 09:36:18 GMT
< Content-Length: 91
< 
* Connection #0 to host localhost left intact
{"error":{"code":403,"status":"Forbidden","message":"invalid csrf token","details":[null]}}
```

## CSRF 拦截器选项
rk-boot 提供了若干 CSRF 拦截器选项，除非是有特殊需要，不推荐覆盖选项。

| 选项 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| gin.interceptors.csrf.enabled	| 启动 CSRF 拦截器 | boolean | false |
| gin.interceptors.csrf.tokenLength | 	Token 长度 | int | 32 |
| gin.interceptors.csrf.tokenLookup | 从哪里获取 Token，请参考下面的介绍  | string | 	“header:X-CSRF-Token” |
| gin.interceptors.csrf.cookieName	| Cookie 名字 | string | _csrf |
| gin.interceptors.csrf.cookieDomain	| Cookie domain | string | "" |
| gin.interceptors.csrf.cookiePath	| Cookie path | string | "" |
| gin.interceptors.csrf.cookieMaxAge	| Cookie MaxAge（秒） | int | 86400 (24小时) |
| gin.interceptors.csrf.cookieHttpOnly	| Cookie HTTP Only 选项 | bool | false |
| gin.interceptors.csrf.cookieSameSite	| Cookie SameSite 选项, 支持 [lax, strict, none, default] | string | "lax" |
| gin.interceptors.csrf.ignorePrefix	| 忽略 CSRF 验证的 Restful API Path | []string | [] |

### tokenLookup 格式
目前支持如下三种方式，拦截器会使用下面方式的一种，在请求中寻找 Token。

- 从 HTTP Header 中获取
- 从 HTTP Form 中获取
- 从 HTTP Query 中获取

```go
// Optional. Default value "header:X-CSRF-Token".
// Possible values:
// - "header:<name>"
// - "form:<name>"
// - "query:<name>"
// Optional. Default value "header:X-CSRF-Token".
```
