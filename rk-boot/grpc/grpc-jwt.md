# gRPC 安全篇-2: 快速实现服务端 JWT 验证

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 实现服务端 JWT 验证逻辑。

> **什么是 JWT？**
>
JSON 网络令牌是一种 Internet 标准，用于创建具有可选签名或可选加密的数据，让两方之间安全地表示声明。令牌使用私有秘密或公共/私有密钥进行签名。

简单来讲，就是通过 JWT 机制，让客户端通过一个密钥，把信息进行加密，添加到 HTTP 请求的 Header中，并传给服务端，服务端验证客户的合法性。
请参考 [JWT 官网](https://jwt.io/)

![image](img/jwt-intro.png)

很多 SAAS 服务，都采用了 JWT 作为用户验证，例如 [github](https://docs.github.com/en/developers/apps/building-github-apps/authenticating-with-github-apps)

**请访问如下地址获取完整教程：**

- https://rkdocs.netlify.app/cn

## 安装
```go 
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
rk-boot 默认会为 gRPC 服务开启 grpc-gateway，两个协议监听同一个端口。

所以，无论是 gRPC 请求还是 Restful 请求，都可以验证 JWT。

### 1.创建 boot.yaml
boot.yaml 文件会告诉 rk-boot 如何启动 gRPC 服务。

在下面的 YAML 文件中，我们声明了三件事：
- 开启 commonService: 默认提供了类似 /rk/v1/healthy 这种 API。[详情](https://github.com/rookie-ninja/rk-grpc#common-service-1)
- 开启 enableRkGwOption: 这个 Option 可以规范 gRPC -> Restful API 的错误类型，否则返回的错误将会是 gRPC 错误码。推荐使用。
- 开启 JWT 拦截器，并声明 JWT 的密钥为 my-secret 作为例子。后台会默认使用 HS256 算法，也可以自定义算法，我们会在下面介绍。

所以，boot.yaml 文件会告诉 rk-boot 开启 8080 端口启动 gRPC 服务，并开启 commonService 以及 JWT 验证拦截器。

```yaml
---
grpc:
  - name: greeter                     # Required
    port: 8080                        # Required
    enabled: true                     # Required
    enableRkGwOption: true            # Optional
    commonService:
      enabled: true                   # Optional, default: false
    interceptors:
      jwt:
        enabled: true                 # Optional, default: false
        signingKey: "my-secret"       # Required
```

### 2.创建 main.go
由于我们开启了 commonService，就不需要额外给 gRPC 添加 API 进行验证了。

```go
// Copyright (c) 2021 rookie-ninja
//
// Use of this source code is governed by an Apache-style
// license that can be found in the LICENSE file.
package rkdemo

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
)

// Application entrance.
func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.创建 JWT Token
根据不同的语言，有很多开源库可以帮助我们创建 JWT Token，请参考[官网](https://jwt.io/libraries)

这里，为了方便，我们直接从[官网](https://jwt.io)里创建一个 Token，这个 Token 使用了 my-secret 作为密钥，HS256 作为算法，与 boot.yaml 里的配置一样。

![image](img/jwt-my-secret.png)

### 4.验证
- 发送请求到 /rk/v1/healthy，并提供上面创建的合法 JWT Token。

```
$ curl localhost:8080/rk/v1/healthy -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.EpM5XBzTJZ4J8AfoJEcJrjth8pfH28LWdjLo90sYb9g"
{"healthy":true}
```

- 发送请求到 /rk/v1/healthy，提供非法 JWT

```
$ curl localhost:8080/rk/v1/healthy -H "Authorization: Bearer invalid-jwt-token"
{
    "error":{
        "code":401,
        "status":"Unauthorized",
        "message":"invalid or expired jwt",
        "details":[
            {
                "code":16,
                "status":"Unauthenticated",
                "message":"[from-grpc] invalid or expired jwt"
            },
            {
                "code":2,
                "status":"Unknown",
                "message":"token contains an invalid number of segments"
            }
        ]
    }
}
```

## JWT 拦截器选项
rk-boot 提供了若干 JWT 拦截器选项，除非是有特殊需要，不推荐覆盖选项。

| 选项 | 描述 | 类型 | 默认值 |
| --- | --- | --- | --- |
| grpc.interceptors.jwt.enabled	| 启动 JWT 拦截器 | boolean | false |
| grpc.interceptors.jwt.signingKey | 	必要, signing key | string | "" |
| grpc.interceptors.jwt.ignorePrefix | 不验证特定 gRPC method 的 JWT | string | "" |
| grpc.interceptors.jwt.signingKeys	| 提供多个 singing Key，请参考下面的介绍 | []string | [] |
| grpc.interceptors.jwt.signingAlgo	| 签名算法，rk-boot 默认使用了 golang-jwt/jwt 作为依赖，支持的算法类型请参考下面的介绍 | string | HS265 |
| grpc.interceptors.jwt.tokenLookup	| 拦截器寻找 CSRF Token 的方法，请参考下面的介绍 | string | “header:Authorization” |
| grpc.interceptors.jwt.authScheme	| Auth Scheme，也就是 Authorization header 后面跟着的 Scheme | string | Bearer |

### 支持的签名算法
HS256，HS384，HS512，RS256，RS384，RS512，ES256，ES384，ES512，EdDSA

### tokenLookup 格式
gRPC 只支持从 Header 里读取，如果是 gRPC 协议发出来的请求，Header 就是 grpc.metadata。

```go
// Optional. Default value "header:Authorization".
// Possible values:
// - "header:<name>"
// Multiply sources example:
// - "header: Authorization"
```

### 多种 signing keys
请参考 [RFC7515](https://www.rfc-editor.org/rfc/rfc7515#section-4.1.4)

提供多个 signing keys，用 : 分开 Key 和 Value。

```yaml
signingKeys:
  - "key:value"
```

### gRPC 协议 JWT 例子
前面的例子中，我们使用了 Restful API 作为请求例子，这次，我们使用 gRPC。

还是启动同样的 gRPC 服务，这次我们使用 [grpcurl](https://github.com/fullstorydev/grpcurl) 来直接调用 gRPC 服务。

为了能使用 grpcurl，我们需要在 boot.yaml 中通过 **enableReflection** 开启 grpc reflection。

- boot.yaml

```yaml
---
grpc:
  - name: greeter                     # Required
    port: 8080                        # Required
    enabled: true                     # Required
    enableRkGwOption: true            # Optional
    enableReflection: true            # Optional
    commonService:
      enabled: true                   # Optional, default: false
    interceptors:
      jwt:
        enabled: true                 # Optional, default: false
        signingKey: "my-secret"       # Required
```

- 发送请求到 rk.api.v1.RkCommonService.Healthy，并提供合法 JWT Token。

```
$ grpcurl -plaintext -H "Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.EpM5XBzTJZ4J8AfoJEcJrjth8pfH28LWdjLo90sYb9g" localhost:8080 rk.api.v1.RkCommonService.Healthy
{
    "healthy": true
}
```

- 发送请求到 /rk/v1/healthy，提供非法 JWT

```
$ grpcurl -plaintext -H "Authorization:Bearer invalid-jwt-token" localhost:8080 rk.api.v1.RkCommonService.Healthy
Error invoking method "rk.api.v1.RkCommonService.Healthy": rpc error: code = Unauthenticated desc = failed to query for service descriptor "rk.api.v1.RkCommonService": invalid or expired jwt
```
