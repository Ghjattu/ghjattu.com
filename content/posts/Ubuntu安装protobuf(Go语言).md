---
title: "Ubuntu20.04安装Protobuf并实现一个简单Go程序"
date: 2023-05-28T19:05:44+08:00
lastmod: '2023-05-29'
author: 'Ghjattu'
slug: 'install-protobuf-in-ubuntu'
categories: ['Linux']
description: "在操作系统为Ubuntu20.04的云服务器ECS上安装配置Protocol Buffer，并用Protobuf实现了一个简单的Go语言程序，讲解了protoc命令中的go_out参数。"
tags: ['Go', 'Protobuf']
---

## 安装protoc 

从 [Protobuf Releases](https://github.com/protocolbuffers/protobuf/releases) 下载最新版本的压缩包：

```shell
wget https://github.com/protocolbuffers/protobuf/releases/download/v23.2/protoc-23.2-linux-x86_64.zip
```

然后解压到 `/use/local` 目录下：

```shell
unzip protoc-23.2-linux-x86_64.zip -d /usr/local
```

如果提示 `Command 'unzip' not found.` 的错误，按照提示先安装：

```shell
apt install unzip
```

然后再执行解压操作。

解压完成后执行 `protoc --version` 查看版本，如果能正常显示版本就说明安装成功。

```shell
protoc --version
# libprotoc 23.2
```

## 安装protoc-gen-go

proton-gen-go 是编译器 protoc 的一个插件，它能够通过 `.proto` 文件生成 Go 语言代码。

```shell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

## 一个简单的例子

如果使用 VSCode 编辑器的话推荐一个 protobuf 插件：[vscode-proto3](https://marketplace.visualstudio.com/items?itemName=zxh404.vscode-proto3) ，提供了语法高亮、代码补全、格式化等功能。

创建一个 `studu-go` 文件夹，然后在文件夹内创建一个 `user` 文件夹，在 `user` 文件夹内新建 `user.proto` ：

```protobuf
syntax = "proto3";

package user;
option go_package=".;user";

message User {
    string name = 1;
    string email = 2;
    int32 age = 3;
    repeated string phone = 4;
}
```

第 1 行声明使用 `proto3` 版本，不写的话默认是 `proto2` 版本。

第 3 行声明 proto 文件所在的包名，相当于 proto 文件的命名空间，防止不同的消息类型产生命名冲突，该字段是可选的。

第 4 行声明了要生成的 `.pb.go` 文件保存的路径（这个放在后面生成代码时讲）和 `.pb.go` 文件的包名，用分号 `;` 隔开。

第 6 行为想要编码的数据结构定义了一个 `message` 类型，其中每个字段后面的 `=1, =2` 表示字段在二进制编码时使用的唯一标记，关键字 `repeated` 表示该数据可重复，类比 Go 语言中的 slice 。

完整的语法可以查询官方文档：[Protocol Buffers Documentation](https://protobuf.dev/programming-guides/proto3/) 。

接着在 `/user` 目录下生成 Go 代码：

```shell
# /study-go/user/
protoc --go_out=. *.proto
```

这会在 `/user` 目录下生成 `user.pb.go` 文件。

`protoc` 命令有两个重要参数：`protoc --go_out=$DST_DIR $SRC_DIR/<name>.proto` ：

第二个参数指定 `.proto` 文件的路径，这个很好理解。

第一个参数 `go_out` 表示 Go 代码输出的目录，**如果在 `.proto` 文件中定义了 `go_package=$PATH`， 那么最终 Go 代码输出的目录是 `$DST_DIR/$PATH`** ，下面举几个例子说明：

当前的文件结构是这样的：

```
/study-go/
├── go.mod
├── go.sum
├── main.go
└── user
    └── user.proto
```

1. 如果在 `/study-go` 目录下执行：

   ```shell
   #  go_package=".;user"
   #  ~/study-go
   protoc --go_out=. ./user/user.proto
   ```

   这样定义的输出路径是 `.` 也即当前目录，最终会在 `/study-go` 目录下生成 `user.pb.go` 。

2. 如果在 `/study-go` 目录下执行：

   ```shell
   #  go_package="/user;user"
   #  ~/study-go
   protoc --go_out=. ./user/user.proto
   ```

   这样定义的输出路径是 `./user` ，也即 `/study-go/user` 。

3. 如果在 `/study-go/user` 目录下执行：

   ```shell
   #  go_package=".;user"
   #  ~/study-go/user
   protoc --go_out=. ./user.proto
   ```

   这样定义的输出路径也是 `.` 也即当前目录，最终会在 `/study-go/user` 目录下生成 `user.pb.go` 。

总的来说，最后输出的路径和当前执行命令的目录、`go_out` 参数、`go_package` 都有关系，官方推荐尽量写在 `go_package` 中，这样可以缩短 `protoc` 命令的长度。

弄清楚生成路径之后，就可以编写 `main.go` 来实现一个简单的程序了：

```go
package main

import (
	"fmt"
	"study-go/user"

	"google.golang.org/protobuf/proto"
)

func main() {
	u := &user.User{
		Name:  "alice",
		Email: "example@com",
		Age:   20,
		Phone: []string{"123", "456"},
	}
	data, _ := proto.Marshal(u)
	userTest := &user.User{}
	proto.Unmarshal(data, userTest)
  fmt.Println(userTest.Name) // output: alice
}
```

在主函数中定义个一个 `User` 类型的变量，然后使用 `data` 变量保存 `proto.Marshal` 序列化产生的数据，最后用 `proto.Unmarshal` 反序列化，最后的 `userTest` 会和 `u` 有相同的数据。
