---
title: "Golang中的接口interface(二)"
date: 2023-05-19T15:40:21+08:00
lastmod: '2023-05-19'
author: 'Ghjattu'
slug: 'golang-interface-part-2'
description: '这篇文章简要地介绍了Golang中接口的概念、接口的方法集、不可寻址类型和可寻址类型和接口嵌套'
categories: ["Golang"]
tags: ['Go', 'interface']
---

## 方法集

在 [Golang中的interface(一)](https://ghjattu.github.io/posts/golang-interface-part-1/) 这一篇文章中，我们实现一个接口都使用的值接收者方法，如果换成指针接收者会怎么样呢？

```go
package main

import "fmt"

type Describer interface {
	Describe()
}

type Person struct {
	Name string
	Age  int
}

func (p *Person) Describe() {
	fmt.Printf("%s is %d years old\n", p.Name, p.Age)
}

func main() {
	p := Person{"Alice", 20}
	var d Describer = p
	d.Describe()
}
```

如果运行上面的代码，会产生一个编译错误：

```
cannot use p (variable of type Person) as Describer value in variable declaration: Person does not implement Describer (method Describe has pointer receiver)
```

它指出，`Person` 类型的值 `p` 没有实现 `Describer` 接口，因为 `Describe` 方法使用指针接收者。而我们在学习结构体方法的时候了解到，指针接收者方法允许值调用，Go 语言会在背后执行 `&` 取地址操作，为什么上述代码会编译失败呢？在解释这个问题之前，我们需要了解**方法集**。

方法集定义了一组关联到给定类型的值或指针的方法，根据实现方法使用的接收者类型，决定该方法是关联到值，还是关联到指针，还是两者都关联。

首先介绍一下 Go 语言规范里定义的方法集规则，如下表格：

| Values |   Method Receivers   |
| :----: | :------------------: |
|  `T`   |       `(t T)`        |
|  `*T`  | `(t T)` and `(t *T)` |

规则中说到，`T` 类型的值的方法集只包含使用值接收者实现的方法；而指向 `T` 类型的指针的方法集既包括使用值接收者实现的方法，也包括使用指针接收者实现的方法。

如果我们换一个角度，以接收者的角度来看上述规则，就是下面的表格：

| Method Receivers |    Values    |
| :--------------: | :----------: |
|     `(t T)`      | `T` and `*T` |
|     `(t *T)`     |     `*T`     |

这个表格可以这样解释：如果用值接收者实现接口的一个方法，那么这个方法会关联到类型 `T` 的值和指针；如果用指针接收者实现接口的一个方法，那么这个方法只会关联到类型 `T` 的指针。

因此，回到上面编译失败的问题，只需要把赋值语句改成：

```go
var d Describer = &p
```

就可以编译通过并输出正确结果。

现在又有了一个新的问题，为什么会有这种限制？事实上，编译器并不是总能获取一个值的地址，保存在一个接口里的动态值是**不可寻址**的，因此有了文章开头的编译错误信息。一种对于动态值不可寻址的解释是：一个接口可以由多种类型实现，假设一开始接口的动态值保存了类型 `A` 的值，并且创建了一个类型 `A` 的指针 `ptr` 指向动态值的地址，若之后将接口的动态值设置为类型 `B` 的值，此时 `ptr` 不再是一个有效的指针了。

有关可寻址类型和不可寻址类型的更多信息可以查看参考资料中的 2、3、4 。

## 接口嵌套

Go 语言不支持继承，但支持接口嵌套，此时，一个接口可以嵌入其他接口，或将一个接口的方法签名嵌入另一个接口中。下面是一个例子：

```go
type SayHello interface {
	Hello()
}

type SayHi interface {
	Hi()
}

type Greeting interface {
	SayHello
	SayHi
}
```

