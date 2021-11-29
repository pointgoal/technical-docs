# Echo 框架：开启 TLS/SSL

## 介绍
通过一个完整例子，在 Echo 框架中开启 TLS/SSL，我就是我们常说的 https。

我们将会使用 [rk-boot](https://github.com/rookie-ninja/rk-boot) 来启动 Echo 框架的微服务。

请访问如下地址获取完整教程：

- https://rkdocs.netlify.app/cn

## 生成 Self-Signed Certificate
用户可以从各大云厂商购买证书，或者使用 [cfssl](https://github.com/cloudflare/cfssl) 创建自定义证书。

我们介绍如何在本地生成证书。

### 1.下载 cfssl & cfssljson 命令行
> 推荐使用 rk 命令行来下载。

```shell script
$ go get -u github.com/rookie-ninja/rk/cmd/rk
$ rk install cfssl
$ rk install cfssljson
```

> 官网下载

```shell script
$ go get github.com/cloudflare/cfssl/cmd/cfssl
$ go get github.com/cloudflare/cfssl/cmd/cfssljson
```

### 2.生成 CA
```shell script
$ cfssl print-defaults config > ca-config.json
$ cfssl print-defaults csr > ca-csr.json
```

根据需要修改 ca-config.json 和 ca-csr.json。

```shell script
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

### 3.生成服务端证书
**server.csr**，**server.pem** 和 **server-key.pem** 将会被生成。

```shell script
$ cfssl gencert -config ca-config.json -ca ca.pem -ca-key ca-key.pem -profile www ca-csr.json | cfssljson -bare server
```

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
[rk-boot](https://github.com/rookie-ninja/rk-boot) 支持通过如下方式让 gRPC 服务获取证书。

- 本地文件系统
- 远程文件系统
- Consul
- ETCD

我们先看看如何从本地获取证书并启动。

### 1.创建 boot.yaml
在这个例子中，我们只启动服务端的证书。其中，locale 用于区分不同环境下 cert。

请参考之前的文章了解详情：

```yaml
---
cert:
  - name: "local-cert"                     # Required
    provider: "localFs"                    # Required, etcd, consul, localFs, remoteFs are supported options
    locale: "*::*::*::*"                   # Required, default: ""
    serverCertPath: "cert/server.pem"      # Optional, default: "", path of certificate on local FS
    serverKeyPath: "cert/server-key.pem"   # Optional, default: "", path of certificate on local FS
echo:
  - name: greeter
    port: 8080
    enabled: true
    enableReflection: true
    cert:
      ref: "local-cert"                    # Enable grpc TLS
    commonService:
      enabled: true
```

### 2.创建 main.go 
```go
package main

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

### 3.文件夹结构 

```shell script
.
├── boot.yaml
├── cert
│   ├── server-key.pem
│   └── server.pem
├── go.mod
├── go.sum
└── main.go

1 directory, 6 files
```

### 4.启动 main.go
```shell script
$ go run main.go
```

### 5.验证
```shell script
$ curl -X GET --insecure https://localhost:8080/rk/v1/healthy                 
{"healthy":true}
```

## 架构
![image](img/tls-arch.png)

## 参数介绍
### 1.从本地读取证书
| 配置项 | 详情 | 需要 | 默认值 |
| ------ | ------ | ------ | ------ |
| cert.localFs.name | 本地文件系统获取器名称 | 是 | "" |
| cert.localFs.locale | 遵从 locale: \<realm\>::\<region\>::\<az\>::\<domain\> | 是 | "" | 
| cert.localFs.serverCertPath | 服务器证书路径 | 否 | "" |
| cert.localFs.serverKeyPath | 服务器证书密钥路径 | 否 | "" |
| cert.localFs.clientCertPath | 客户端证书路径 | 否 | "" |
| cert.localFs.clientCertPath | 客户端证书密钥路径 | 否 | "" |

- 例子

```yaml
---
cert:
  - name: "local-cert"                     # Required
    description: "Description of entry"    # Optional
    provider: "localFs"                    # Required, etcd, consul, localFs, remoteFs are supported options
    locale: "*::*::*::*"                   # Required, default: ""
    serverCertPath: "cert/server.pem"      # Optional, default: "", path of certificate on local FS
    serverKeyPath: "cert/server-key.pem"   # Optional, default: "", path of certificate on local FS
echo:
  - name: greeter
    port: 8080
    enabled: true
    enableReflection: true
    cert:
      ref: "local-cert"                    # Enable grpc TLS
```

### 2.从远程文件服务读取证书
| 配置项 | 详情 | 需要 | 默认值 |
| ------ | ------ | ------ | ------ |
| cert.remoteFs.name | 远程文件服务获取器名称 | 是 | "" |
| cert.remoteFs.locale | 遵从 locale：\<realm\>::\<region\>::\<az\>::\<domain\> | 是 | "" | 
| cert.remoteFs.endpoint | 远程地址： http://x.x.x.x 或者 x.x.x.x | 是 | N/A |
| cert.remoteFs.basicAuth | Basic auth： <user:pass>. | 否 | "" |
| cert.remoteFs.serverCertPath | 服务器证书路径 | 否 | "" |
| cert.remoteFs.serverKeyPath | 服务器证书密钥路径	| 否 | "" |
| cert.remoteFs.clientCertPath | 客户端证书路径 | 否 | "" |
| cert.remoteFs.clientCertPath | 客户端证书密钥路径 | 否 | "" |

- 例子

```yaml
---
cert:
  - name: "remote-cert"                    # Required
    description: "Description of entry"    # Optional
    provider: "remoteFs"                   # Required, etcd, consul, localFs, remoteFs are supported options
    endpoint: "localhost:8081"             # Required, both http://x.x.x.x or x.x.x.x are acceptable
    locale: "*::*::*::*"                   # Required, default: ""
    serverCertPath: "cert/server.pem"      # Optional, default: "", path of certificate on local FS
    serverKeyPath: "cert/server-key.pem"   # Optional, default: "", path of certificate on local FS
echo:
  - name: greeter
    port: 8080
    enabled: true
    cert:
      ref: "remote-cert"                   # Enable grpc TLS
```

### 3.从 Consul 读取证书
| 配置项 | 详情 | 需要 | 默认值 |
| ------ | ------ | ------ | ------ |
| cert.consul.name | Consul 获取器名称 | 是 | "" |
| cert.consul.locale | 遵从 locale: \<realm\>::\<region\>::\<az\>::\<domain\> | 是 | "" | 
| cert.consul.endpoint | Consul 地址： http://x.x.x.x or x.x.x.x | 是 | N/A |
| cert.consul.datacenter | Consul 数据中心 | 是 | "" |
| cert.consul.token | Consul 访问密钥 | 否 | "" |
| cert.consul.basicAuth | Consul Basic auth，格式：<user:pass>. | 否 | "" |
| cert.consul.serverCertPath | 服务器证书路径	 | 否 | "" |
| cert.consul.serverKeyPath | 服务器证书密钥路径 | 否 | "" |
| cert.consul.clientCertPath | 服务器证书密钥路径 | 否 | "" |
| cert.consul.clientCertPath | 服务器证书密钥路径 | 否 | "" |

- 例子

```yaml
---
cert:
  - name: "consul-cert"                    # Required
    provider: "consul"                     # Required, etcd, consul, localFS, remoteFs are supported options
    description: "Description of entry"    # Optional
    locale: "*::*::*::*"                   # Required, ""
    endpoint: "localhost:8500"             # Required, http://x.x.x.x or x.x.x.x both acceptable.
    datacenter: "dc1"                      # Optional, default: "", consul datacenter
    serverCertPath: "server.pem"           # Optional, default: "", key of value in consul
    serverKeyPath: "server-key.pem"        # Optional, default: "", key of value in consul
echo:
  - name: greeter
    port: 8080
    enabled: true
    cert:
      ref: "consul-cert"                   # Enable grpc TLS
```

### 4.从 ETCD 读取证书
| 配置项 | 详情 | 需要 | 默认值 |
| ------ | ------ | ------ | ------ |
| cert.etcd.name | ETCD 获取器名称 | 是 | "" |
| cert.etcd.locale | 遵从 locale:  \<realm\>::\<region\>::\<az\>::\<domain\> | 是 | "" | 
| cert.etcd.endpoint | ETCD 地址：http://x.x.x.x or x.x.x.x | 是 | N/A |
| cert.etcd.basicAuth | ETCD basic auth，格式：<user:pass>. | 否 | "" |
| cert.etcd.serverCertPath | 服务器证书路径 | 否 | "" |
| cert.etcd.serverKeyPath | 服务器证书路径 | 否 | "" |
| cert.etcd.clientCertPath | 客户端证书路径 | 否 | "" |
| cert.etcd.clientCertPath | 客户端证书密钥路径 | 否 | "" |

- 例子

```yaml
---
cert:
  - name: "etcd-cert"                      # Required
    description: "Description of entry"    # Optional
    provider: "etcd"                       # Required, etcd, consul, localFs, remoteFs are supported options
    locale: "*::*::*::*"                   # Required, default: ""
    endpoint: "localhost:2379"             # Required, http://x.x.x.x or x.x.x.x both acceptable.
    serverCertPath: "server.pem"           # Optional, default: "", key of value in etcd
    serverKeyPath: "server-key.pem"        # Optional, default: "", key of value in etcd
echo:
  - name: greeter
    port: 8080
    enabled: true
    cert:
      ref: "etcd-cert"                   # Enable grpc TLS
```
