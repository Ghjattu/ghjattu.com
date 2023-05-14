---
title: "Golang中有关slice的知识点"
date: 2023-05-14T20:08:46+08:00
author: "Ghjattu"
slug: 'slice-in-golang'
categories: ["golang"]
---

## slice的内部实现

slice不存储任何数据，它只描述了底层数组中的一段，它是一个包含三个字段的数据结构：

1. 指针：指向slice第一个元素对应的底层数组元素的地址
2. 长度：即slice中能访问的元素的个数
3. 容量：即slice中第一个元素对应的底层数组元素 到 数组末尾元素的个数

可以把 slice 想象成下面的结构：

```go
type slice struct {
	Length   int
	Capacity int
	Ptr      *int
}
```

例如：对于底层数组容量是 $k$ 的 `slice[i:j]` 来说，长度为 $j-i$ ，容量为 $k-i$ 。

因为slice包含指向slice第一个元素的指针，因此向函数传递slice将允许在函数内部修改底层数组的元素。

多个slice可以共享一个底层数组，对其中一个slice的修改会反映到其他slice上。

## slice的创建和初始化

一种创建 slice 的方法是内置的 make 函数，make 函数会分配一个元素为零值的底层数组并返回一个引用了它的 slice，如果只指定了长度，那么 slice 的长度和容量相等：

```go
s := make([]int, 3) 
// len(s) = 3, cap(s) = 3
```

也可以传入第三个参数指定容量：

```go
s := make([]int, 3, 5)
// len(s) = 3, cap(s) = 5
```

另一种创建 slice 的方法是 slice 字面量，和创建数组类似，只是不需要指定 $[]$ 中的值，长度和容量基于初始化给定的元素个数决定：

```go
s := []int{1, 2, 3}
// len(s) = 3, cap(s) = 3
```

使用一个大于等于 `len(s)` 的索引访问 slice 元素会引发一个panic：

```go
s := make([]int, 3, 5)
s[3] = 1
// panic: runtime error: index out of range [3] with length 3
```

## nil和空slice

一个零值的 slice 等于nil，一个为 nil 的 slice 没有底层数组，长度和容量均为 0 ：

```go
var s []int      // s == nil
s := []int(nil)  // s == nil
```

声明一个空 slice 也不会分配存储空间，长度和容量也均为 0 ：

```go
s := make([]int, 0) // s != nil
s := []int{}        // s != nil
```

如果需要判断一个 slice 是否为空，应该使用 `len(s) == 0` ，而**不应该**使用 `s == nil` 。

不管是 nil 值的 slice 还是空 slice，除了文档明确说明的地方，所有函数应该以相同的方式对待 nil 值的 slice和空 slice，对其调用 `append`、`len` 和 `cap` 的效果都是一样的。

## slice的增长

调用函数 `append` 会返回一个包含修改结果的新 slice，`append` 总是会增加新 slice 的长度，而容量可能改变，也可能不改变，这取决于被操作的 slice 的可用容量。

如果底层数组没有足够的容量，`append` 会创建一个新的底层数组，然后把旧数组的值复制进来，再追加新的值。

```go
s := []int{1, 2, 3, 4}
news := s

fmt.Println(&s[0])      // 0x140000e8000
fmt.Println(&news[0])   // 0x140000e8000
news = append(news, 5)
fmt.Println(&s[0])      // 0x140000e8000
fmt.Println(&news[0])   // 0x140000a8100
```

## slice的迭代

关键字 range 可以配合关键字 for 来迭代 slice 中的元素，range 会返回两个值：第一个是当前迭代到的索引位置，第二个是当前迭代到的元素的**副本**。range 创建了每个元素的副本，而不是直接返回对该元素的引用，如果使用该值变量的地址作为指向每个元素的指针，就会造成错误。

```go
s := []int{1, 2, 3, 4}
for index, value := range s {
  fmt.Printf("value: %d, value-addr: %p, elem-addr: %p\n", value, &value, &s[index])
}

// output
// value: 1, value-addr: 0x1400011a0d8, elem-addr: 0x14000160000
// value: 2, value-addr: 0x1400011a0d8, elem-addr: 0x14000160008
// value: 3, value-addr: 0x1400011a0d8, elem-addr: 0x14000160010
// value: 4, value-addr: 0x1400011a0d8, elem-addr: 0x14000160018
```

## slice的内存优化

slice 指向一个底层数组，只要 slice 一直存在内存中，其底层数组就不会被垃圾回收。假设有一个非常大的底层数组，但我们只关注其中很小的一部分，我们在这个底层数组上建立一个 slice ，然后处理这个 slice 。这时这个底层数组会一直保存在内存中，因为有一个 slice 引用了它。

深拷贝和浅拷贝：对于浅拷贝，复制出的新对象指向的地址和原对象相同，`b:=a` 和 `b:=a[:]` 都是浅拷贝；而深拷贝出的新对象指向地址和原对象不同，例如  [`copy`](https://pkg.go.dev/builtin#copy) 函数：

```go
oldSlice := []int{0, 1, 2, 3, 4, 5, 6, 7}
newSlice := make([]int, len(oldSlice))
copy(newSlice, oldSlice)
fmt.Printf("%p\n", &oldSlice[0])  // 0x140000a8040
fmt.Printf("%p\n", &newSlice[0])  // 0x140000a8080
```

因此，上述问题的一个解决方式是：我们创建一个新的 slice，其长度和容量等于需要处理的数组长度，然后使用 `copy` 函数将元素复制到新 slice 中，后续使用新 slice 进行数据处理，这样原底层数组就会被回收了。

```go
func solve(oldSlice []int) []int {
	need := oldSlice[0:2]
	newSlice := make([]int, len(need))
	copy(newSlice, need)
	return newSlice
}

func main() {
	oldSlice := []int{0, 1, 2, 3, 4, 5, 6, 7}
	newSlice := solve(oldSlice)
	fmt.Printf("%p\n", &oldSlice[0])  // 0x140000a8040
	fmt.Printf("%p\n", &newSlice[0])  // 0x140000a6020
}
```
## 参考资料
1. 《Go in Action》
2. 《The Go Programming Language》
