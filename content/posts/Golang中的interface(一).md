---
title: "Golang中的接口interface(一)"
date: 2023-05-15T15:16:55+08:00
lastmod: '2023-05-15'
author: 'Ghjattu'
slug: 'golang-interface-part-1'
description: '这篇文章简要地介绍了Golang中接口的概念、内部表示以及基本的声明和实现方式'
categories: ["Golang"]
---

## 什么是接口

在 Go 语言中，接口是一组方法签名 (method signatures) 的集合，这些方法的行为不由接口直接实现，如果某个用户定义的类型实现了一个接口中的所有方法，就说明该类型实现了这个接口，这个用户定义的类型的值就可以赋给这个接口类型的值，对接口值方法的调用会执行接口值里存储的用户定义的类型的值对应的方法。

## 定义并实现一个接口

下面是一个简单的示例：

```go
type SayHello interface {
	hello()
}

type Person1 struct { 
	name string
	age  int
}

type Person2 struct { 
	name  string
	email string
}

func (p1 Person1) hello() { 
	fmt.Printf("%s is %d years old\n", p1.name, p1.age)
}

func (p2 Person2) hello() {
	fmt.Printf("%s's email is %s\n", p2.name, p2.email)
} 

func greeting(s []SayHello) { 
	for _, p := range s {
		p.hello()
	}
}

func main() {
	p1 := Person1{"Alice", 20} 
	p2 := Person2{"Bob", "example@gmail.com"}
	s := []SayHello{p1, p2}
	greeting(s)
}
```

第 1 行定义了一个 `SayHello` 接口，里面只有一个 `hello()` 方法，在第 5 行和第 10 行定义了两个结构体类型，然后第 15 行和第 19 行分别使用值接收者 (value receiver) 实现了该方法，此时可以说两个自定义类型 `Person1` 和 `Person2` 均实现了 `SayHello` 接口。在 Go 语言中没有 `implements` 关键字，只要一个类型实现了接口的所有方法，就隐式地实现了接口。

之后，在第 23 行我们创建了一个 `greeting` 函数，它接收一个 `SayHello` 接口的 slice 作为参数。接下来在第 33 行调用 `greeting` 函数，在 `for range` 循环中调用各自的 `hello()` 方法，得到正确的输出：

```
Alice is 20 years old
Bob's email is example@gmail.com
```

假设在未来又定义了新的结构体 `Person3` ，只要结构体实现了 `hello()` 方法，它就可以被加入到 `SayHello` 接口的 slice 中，而不需要修改 `greeting` 函数的代码。

## 接口的内部表示

一个接口值可以被想象成一个二元组 `(type, value)` ，其中 `type` 代表接口值存储的具体的类型信息，`value` 代表接口值存储的具体的值的信息，它们也被称作接口的动态类型和动态值。

下面是一个简单的示例：

```go
type Employee interface {
	GetSalary() int
}

type Person struct {
	name       string
	age        int
	baseSalary int
	rank       int
}

func (p Person) GetSalary() int {
	return p.baseSalary * p.rank
}

func main() {
	p := Person{"Alice", 20, 100, 2}
	var e Employee = p
	fmt.Printf("Interface type: %T,  value: %v\n", e, e)
}
```

第 17 行我们创建了一个 `Person` 类型的实例并把它赋值给一个 `Employee` 类型的变量，第 19 行执行的结果如下：

```
Interface type: main.Person,  value: {Alice 20 100 2}
```

在 Go 语言中，变量总是被一个定义明确的值初始化，即使接口类型也不例外。如果我们只声明了一个接口类型的变量：`var e Employee` ，却没有赋初值，那么它的 `type` 和 `value` 字段均为 nil 。一个接口值依据它的 `type` 被描述为空或非空，所以 `e` 是一个空的接口值，可以通过 `e == nil` 或 `e != nil` 来判断一个接口值是否为空，调用一个空接口值上的任意方法都会产生一个 panic 。

如果 `type` 是不可比较的（比如 slice），那么用 `==` 运算符比较两个接口值会产生一个 panic；否则，两个接口值可以使用 `==` 和 `!=` 来进行比较，两个接口值相等仅当两个接口值均为 nil ，或它们的 `type` 相同且 `value` 也根据 `==` 运算符相等。接口值可比较也代表它们可以作为 `map` 的键。

## 空接口

没有声明任何方法的接口被称作是一个空接口，表示为 `interface{}` 。因为没有方法，所以所有类型都实现了空接口。

空接口被用来处理未知类型的值，比如 `fmt.Print` 可接受任意数量的类型为 `interface{}` 的值。

## 类型断言

类型断言尝试获取一个接口值的动态值，即它的 `value`，语法是 `i.(T)` ，其中 `i` 是一个接口值，`T` 是一个具体的类型。

```go
func main() {
	var i interface{} = 42
	v := i.(int)
	fmt.Println(v)
}
```

上述程序的第 3 行断言接口值 `i` 保存了具体类型 `int` 的值，并把相应的值赋值给变量 `v` ，该程序会输出 42 。

但如果 `i` 没有保存具体类型 `T` 的值，执行 `v := i.(T)` 会产生一个 panic ：

```go
func main() {
	var i interface{} = 42
	v := i.(string)
	fmt.Println(v)
}
```

上述程序会输出：`panic: interface conversion: interface {} is int, not string`

为了解决这个问题，一个更合适的使用语法是：`v, ok := i.(T)` ，这个类型断言返回两个值：如果 `i` 保存了具体类型 `T` 的值，返回对应的值且 `ok` 为 `true` ；否则，`v` 被设置为零值且 `ok` 为 `false` ，不会产生 panic 。

```go
func main() {
	var i interface{} = "Hello"
	v1, ok := i.(string)
	fmt.Println(v1, ok)
	v2, ok := i.(int)
	fmt.Println(v2, ok)
}
```

上述程序输出：

```
Hello true
0 false
```

## 类型选择

类型选择用来把接口值 `i` 的动态类型 `type` 和多个具体类型 `T` 做比较，其语法类似于 `switch case` 语句，区别在于类型选择中的 `case` 为具体类型而非值。

```go
func check(i interface{}) {
	switch i.(type) {
	case int:
		fmt.Println("Type is int")
	case string:
		fmt.Println("Type is string")
	default:
		fmt.Println("Unknown type")
	}
}

func main() {
	check(42)
	check("Hello")
	check(42.42)
}
```

上述程序会输出：

```
Type is int
Type is string
Unknown type
```

也可以在 `case` 语句中使用接口类型，如下面的程序：

```go
type Employee interface {
	GetSalary()
}

type Person struct {
	name       string
	age        int
	baseSalary int
	rank       int
}

func (p Person) GetSalary() {
	fmt.Println(p.baseSalary * p.rank)
}

func check(i interface{}) {
	switch v := i.(type) {
	case Employee:
		v.GetSalary()
	default:
		fmt.Println("Unknown type")
	}
}

func main() {
	p := Person{"Alice", 20, 100, 2}
	check("Alice")
	check(p)
}
```

因为 `Person` 实现了 `Employee` 接口，所以第 18 行的 `case Employee:` 会被选中，然后调用 `GetSalary` 方法，程序输出为：

```
Unknown type
200
```

不过有一点需要注意，程序中每一个 `case` 会被顺序地考虑，当 `case` 中包含多个接口类型时，编写 `case` 语句的顺序就十分重要，因为一个类型可能实现了多个接口，导致多个 `case` 语句同时匹配。

## 参考资料

1. 《The Go Programming Language》
1. [A Tour of Go](https://go.dev/tour/welcome/1) 
