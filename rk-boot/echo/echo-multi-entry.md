# Echo 框架：启动多个端口

## 介绍
本文介绍如何通过 [rk-boot](https://github.com/rookie-ninja/rk-boot) 在一个进程里启动多个 Echo 端口。

> **为什么要启动多个端口？** 
>
> 大部分情况下，我们是不需要的。如果我们希望在一个进程里通过 flag 启动不同端口时，会用到。
>
> 我们会在下面的例子里给出一个场景。

请访问如下地址获取完整教程：https://rkdocs.netlify.app/cn

## 安装
```go
go get github.com/rookie-ninja/rk-boot
```

## 快速开始
### 1.创建 boot.yaml
为了验证，我们启动了 commonService，commonService 里包含了一系列常用 API，例如 /rk/v1/healthy。

```yaml
---
echo:
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

### 2.创建 main.go 
通过 **boot.GetEchoEntry("xxx")** 来获取 EchoEntry 进行 API 注册等操作。

```go
package main

import (
	"context"
	"github.com/rookie-ninja/rk-boot"
)

func main() {
	// Create a new boot instance.
	boot := rkboot.NewBoot()

	// Get alice Entry via:
	//fmt.Println(boot.GetEchoEntry("alice"))

	// Get bob Entry via:
	//fmt.Println(boot.GetEchoEntry("bob"))

	// Bootstrap
	boot.Bootstrap(context.Background())

	// Wait for shutdown sig
	boot.WaitForShutdownSig(context.Background())
}
```

### 3.文件夹结构 
```
$ tree
.
├── boot.yaml
├── go.mod
├── go.sum
└── main.go

0 directories, 4 files
```

### 4.启动 main.go
```
$ go run main.go
```

### 5.验证
- 发送请求到 1949
```
$ curl -X GET localhost:1949/rk/v1/healthy
{"healthy":true}
```

- 发送请求到 2008
```
$ curl -X GET localhost:2008/rk/v1/healthy
{"healthy":true}
```

## 只启动 1949 端口
我们可以通过 --rkset 命令行参数来 disable 掉 2008 端口的 EchoEntry。

boot.yaml 里 echo 是一个数组，所以使用 echo[x] 来表示。

### 1.启动 main.go with --rkset
```
$ go build main.go
$ ./main --rkset "echo[1].enabled=false"
```

### 2.验证
- 发送请求到 1949
```
$ curl -X GET localhost:1949/rk/v1/healthy
{"healthy":true}
```

- 发送请求到 2008
```
$ curl -X GET localhost:2008/rk/v1/healthy
curl: (7) Failed to connect to localhost port 2008: Connection refused
```

## rkset 覆盖 boot.yaml 里的值
boot.yaml 里的值，使用 --rkset 作为参数。

使用**逗号**（注意，是英文逗号）来覆盖多个参数。

| 类型 | 例子 |
| ---- | ---- |
| Map | app.description="This is description" |
| List | echo[0].name="alice",echo[0].port=8081 |
