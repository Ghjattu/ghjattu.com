---
title: "M个goroutine交替输出1到N"
date: 2023-11-27T13:01:39+08:00
lastmod: '2023-11-27'
author: 'Ghjattu'
slug: 'm-goroutine-alternately-output-1-to-n'
categories: ['Golang']
description: "文章讲解了如何利用 channel 通信让 M 个 goroutine 交替输出 1 到 N 的数字。"
tags: ['Go']
---

如何用 M 个 goroutine 利用 channel 通信交替输出 1 到 N 的数字？

输出类似于：
```
goroutine 0: 1
goroutine 1: 2
goroutine 2: 3
goroutine 0: 4
goroutine 1: 5
...
```

## 思路
设 goroutine 的编号为 $0,\ 1,\ \dots,\ M-1$ ，首先我们能想到每个 goroutine 都以 $M$ 为步长来输出，
比如编号为 $0$ 的 goroutine 输出 $1,\ M+1,\ 2M+1,\ \dots$ ，编号为 $1$ 的 goroutine 输出 $2,\ M+2,\ 2M+2,\ \dots$ ，以此类推。
让这些 goroutine 不断的按编号顺序循环输出，就可以实现交替输出的效果。

那么现在的问题是如何使用 channel 来控制这些 goroutine 是按编号顺序输出，而不是随机输出呢？

我们可以使用一个下标从 $0$ 开始且长度同样为 $M$ 的无缓冲 channel 数组来控制这些 goroutine 的输出顺序，下标为 $i$ 的 channel 对应编号为 $i$ 的 goroutine 。

编号为 $i$ 的 goroutine 需要接收到前一个也就是编号为 $(i-1+M)\bmod M$ 的 goroutine 发来的信号才能输出，输出完毕后再向下一个也就是编号为 $(i+1)\bmod M$ 的 goroutine 发送信号，以此类推。

所以控制 goroutine 顺序的核心代码为下面这部分：

```go
// 当前的 goroutine 的编号为 idx
for j := i; j <= n; j += m {
	<-chs[idx]
	fmt.Printf("goroutine %d: %d\n", idx, j)
	chs[(idx+1)%m] <- struct{}{}
}
```

这个 `for` 循环就是以 $M$ 为步长来输出数字，在 `for` 循环中，编号为 $idx$ 的 goroutine 首先尝试从对应的 channel 中读数据，当前一个 goroutine 没有发送数据时，这个 goroutine 就会被阻塞；当前 goroutine 输出完毕后，就向下一个 goroutine 对应的 channel 里发送数据，唤醒下一个 goroutine 。

这个算法可以控制 goroutine 顺序输出，但仍存在一个问题：假设最后一个输出数字的 goroutine 的编号为 $idx$ ，那么它执行 `chs[(idx+1)%m] <- struct{}{}` 时，会被永久阻塞。因为此时其他的 $M-1$ 个 goroutine 都已经完成任务并退出了，意味着没有 goroutine 会从该 channel 中读取数据。

通过手动模拟，我们可以发现最后一个 goroutine 会向编号为 $N\bmod M$ 的 channel 发送数据，所以我们在创建 goroutine 的时候需要写个 `if` 判断一下。

或者将 channel 的缓冲区长度设置为 $1$ ，这样最后一个 goroutine 执行 `chs[(idx+1)%m] <- struct{}{}` 时，不会被阻塞，而是将数据写入 channel 的缓冲区，然后退出，这样就不需要写 `if` 判断了。

## 完整代码
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 初始化
	var wg sync.WaitGroup
	n := 20
	m := 3
	wg.Add(m)
	chs := make([]chan struct{}, m)
	for i := 0; i < m; i++ {
		chs[i] = make(chan struct{})
	}

	// 创建 M 个 goroutine
	for i := 1; i <= m; i++ {
		go func(i, idx int) {
			defer wg.Done()
			// 以 M 为步长输出
			for j := i; j <= n; j += m {
				<-chs[idx]
				fmt.Printf("goroutine %d: %d\n", idx, j)
				chs[(idx+1)%m] <- struct{}{}
			}
			if n%m == idx {
				<-chs[idx]
			}
		}(i, i-1)
	}

	// 启动第一个 goroutine
	chs[0] <- struct{}{}

	wg.Wait()
}
```
