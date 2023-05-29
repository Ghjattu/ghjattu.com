---
title: "Ubuntu 20.04安装Golang并使用VSCode远程连接"
date: 2023-05-26T12:45:53+08:00
lastmod: '2023-05-28'
author: 'Ghjattu'
slug: 'install-golang-in-ubuntu'
categories: ['Golang']
description: "在操作系统为Ubuntu20.04的云服务器ECS上安装配置Golang，并在本地使用VSCode远程连接。"
tags: ['Go', 'Linux']
---

为了更好地学（瞎）习（搞）Linux 命令，今天我去阿里云用学生优惠白嫖了一台 ECS ，这篇文章记录一下从连接 ECS 到安装 Go 最后用 VSCode 编写代码的过程和遇到的问题。

## 远程连接ECS

这一步遇到的问题是：在管理台点击远程连接输入密码后会遇到一个服务器不允许密码登陆的错误，同时错误下方也会给出若干排查方法，主要是开启 root 用户远程登录，参考这篇文章解决：[通过密码或密钥认证登录Linux实例](https://help.aliyun.com/document_detail/147650.html?#section-m1n-unh-gr1) 。

然后使用一个 SSH 客户端输入公网 IP 和密码登陆，如果显示 `Welcome to Alibaba Cloud Elastic Compute Service !` 就说明登陆成功了。

## 安装Golang

这一部分的主要目标就是在 `/usr/local` 目录下解压安装包，下面的过程参考了官方文档，首先通过 `pwd` 和 `ls` 命令来定位寻找正确的路径：

```shell
cd usr/local
# 下载
wget -c https://studygolang.com/dl/golang/go1.20.4.linux-amd64.tar.gz
# 解压
tar -xzf go1.20.4.linux-amd64.tar.gz
```

>我使用的 ECS 登陆后默认在 `/root` 文件夹下，而 `usr` 和 `root` 文件夹是同级的，所以可能需要先 `cd ..` 前往上层目录。

然后执行 `ls` 命令就能看到多了一个 `go` 的文件夹。

接下来就是把 `/usr/local/go/bin` 加到环境变量中：

```shell
cd ~
vim .profile
```

把下面几行加入到 `.profile` 文件中：

```shell
export GOPATH="/root/go"
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
```

保存并退出，重新加载环境变量：

```shell
source .profile
```

现在执行 `go version` 看到版本信息 `go version go1.20.4 linux/amd64` 就说明安装成功了。

执行 `go env` 可以看到相关的环境变量，例如 `GOMODULE` 和 `GOPROXY` 都可以设置成和本地一致。 

## VSCode连接

在插件里搜索 Remote SSH，安装后会在左侧菜单栏里出现一个远程资源管理器的图标，按照下图中的顺序，先点击图标，在点击“添加”，然后在弹出的输入框内按照提示的格式输入用户名和 IP 地址，按“Enter”保存到 config 文件中。

<img src="./config.png" style="zoom:35%;" />

然后在远程资源管理器中刷新一下就能看到添加的服务器的地址，点击服务器然后连接，成功后会在 VSCode 上方要求输入密码，验证通过后就可以像在本地一样写代码了。

想关闭连接的话就点击左下角的 SSH：xxx.xxx.xxx，在弹出的对话框中点击关闭就OK了。

## 使用密钥登陆服务器

每次手动输入密码太麻烦了，密钥登陆是更好的解决方案。

SSH 密钥登陆的大致过程如下：

1. 用户在客户端使用 `ssh-keygen` 命令生成私钥和公钥
2. 用户将公钥上传到服务器的指定位置
3. 客户端向服务器发起登陆请求，服务器收到后返回一段随机数据
4. 客户端用私钥加密随机数据再发给服务器
5. 服务器收到后用公钥解密，如果和原始数据一致即身份验证通过

下面生成一对密钥：

```shell
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/username/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
```

第一个问题询问密钥的保存路径，一般默认即可；第二个问题询问私钥的密码，设置了的话即使其他人拿到了私钥也需要输入密码验证身份，为了方便可以直接留空，按回车键即可。

最后生成了两个文件：`ssh/id_rsa` 是私钥，`ssh/id_rsa.pub` 是公钥。

可以用下面两个命令修改密钥的权限，防止其他用户读取：

```shell
chmod 600 ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa.pub
```

接下来把公钥上传到服务器，OpenSSH 规定公钥需要保存到服务器 `~/.ssh/authorized_keys` 文件中，同时也提供了一个命令 `ssh-copy-id` 命令来自动上传：

**注意：** `ssh-copy-id` 命令会将客户端的公钥上传到服务器的 `authorized_keys` 文件末尾，如果之前文件不为空的话，务必确定文件末尾是一个换行符。

```shell
ssh-copy-id -i id_rsa username@host
```

接着会要求输入服务器的登陆密码，正常输入即可。之后使用 `ssh username@host` 命令就可以不用密码了，为了安全还可以进一步关闭服务器的密码登陆。
