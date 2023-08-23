---
title: 尝试在工程中使用 Go 泛型
description: Go 从 1.18 版本开始支持泛型。一起看看怎么回事。
date: 2023-08-14 11:41:17
index_img: img/gopher.png
tags: 
    - Go
    - Generics
categories:
    - Go
---

# 前言

Go 从 1.18 版本开始支持泛型。一起看看怎么回事。


# 从 max、min、sum 开始

找出一个 slice 的最大值、最小值和对一个 slice 求和是很常见的操作，通常我们需要对每种类型的 slice 的都单独实现一套函数操作：

```go
func MaxOfInts(values []int) int { ... }
func MaxOfFloats(values []float32) float32 { ... }

func MinOfInts(values []int) int { ... }
func MinOfFloats(values []float32) float32 { ... }

func SumInts(values []int) int { ... }
func SumFloats(values []float32) float32 { ... }
```

甚至里面的代码都是一样的：

```go
func SumInts(values []int) int {
    var s = 0
    for _, v := range values {
        s += v
    }
    return s
}

func SumFloats(values []float32) float32 {
    var s = 0
    for _, v := range values {
        s += v
    }
    return s
}
```

本着 DRY（Don't Repeat Yourself） 原则，有些人试图使用其他方法来解决这个问题，比如使用反射、代码生成器。反射复杂且性能一般，代码生成器又需要引入新的依赖。

在 Go 推出泛型后带来了新的方案，通过泛型，我们的求和函数支持处理 `[]int` 和 `[]float`：

```go
func Sum[T int | float32](values []T) T {
    var s T
    for _, v := range values {
        s += v
    }
    return s
}

func main() {
    fmt.Printf("sum ints: %d\n", Sum([]int{1, 2, 3, 4, 5})) // sum ints: 15
    fmt.Printf("sum floats: %.2f\n", Sum([]float32{1.1, 2.2, 3.3, 4.4, 5.5})) // sum floats: 16.50
}
```

仔细看泛型函数 Sum 的定义：

```go
func Sum[T int | float32](values []T) T
```

与一般的函数相比，在函数名的后面多了用方括号的这一段，称为类型参数（type parameters）：

```go
[T int | float32]
```

其中 `T` 为类型参数名称、`int | float32` 为类型参数约束，代表 `T` 允许是 `int` 或者 `float32`。


根据上面的例子，泛型版本 `Min`、`Max` 也手到擒来：

```go
func Min[T int | float32](v T, values ...T) T {
	min := v

	for i := range values {
		if values[i] < min {
			min = values[i]
		}
	}

	return min
}

func Max[T int | float32](v T, values ...T) T {
	max := v

	for i := range values {
		if values[i] > max {
			max = values[i]
		}
	}

	return max
}
```


# 类型参数约束

类型约束定义了允许作为类型参数的一组类型。

```go
func Min[T int | float32](v T, values ...T) T
```

比如这里 T 的约束，就是 `int | float32`，允许使用 `int` 或者 `float32` 作为类型参数。


假设我们需要让 `Min`、`Max`、`Sum` 这几个泛型函数传入支持 `string` 类型，我们可以这样做：

```go
func Min[T int | float32 | string](v T, values ...T) T { ... }
func Max[T int | float32 | string](v T, values ...T) T { ... }
func Sum[T int | float32 | string](values []T) T { ... }
```

## 使用 interface 定义一组类型

我们修改了三个地方，使得这三个函数支持传入 `string` 类型，有没有更好的方法呢，答案是有的。

在过去 `interface` 用于定义一组方法，现在 `interface` 可以用于定义一组类型：

```go
type MyTypeSet interface {
    int | float32 | string
}
```

我们可以将它用于类型约束：

```go
func Min[T MyTypeSet](v T, values ...T) T { ... }
func Max[T MyTypeSet](v T, values ...T) T { ... }
func Sum[T MyTypeSet](values []T) T { ... }
```

如果后续我们还需要支持其他类型，就可以直接修改 `MyTypeSet` 即可。

另外 `[T int | float32 | string]` 其实是 `[T interface { int | float32 | string }]` 的语法糖，允许忽略 `interface{}` 使得结构紧凑。

## 支持类型别名

我们定义一个 `MyInt` 作为 `int` 的别名，尝试在 `Sum` 中使用它：

```go
type MyInt int

...

var data []MyInt = ...
var sumOfData = Sum(data)
```

编译会提示如下错误：

```
MyInt does not implement MyTypeSet (possibly missing ~ for int in constraint MyTypeSet)
```

这里提示我们可能在 `MyTypeSet` 约束中的 `int` 缺少 `~` 符号。修改约束，添加上 `~` 符号：

```go
type MyTypeSet interface {
    ~int | float32 | string
}
```

再次编译，没有问题。那么 `~` 符号的作用是什么呢？我们看看官方文档的定义：

> ~T notation stands for “all types that have the underlying type T”

~T 符号表示“具有底层类型 T 的所有类型”。

那么 `~int` 就表示底层类型为 `int` 的所有类型，自然就可以处理 `MyInt`。

## 常用的内建的类型约束

- any：interface{} 的别名，等于没有约束
- comparable：类型支持 `==` 和 `!=` 操作

另外 `golang.org/x/exp` 定义了一组有用的类型约束，点击这里查看：[constraints](https://pkg.go.dev/golang.org/x/exp/constraints)


# 一些例子

## 简单实现一个泛型 Set

```go
type Set[T comparable] struct {
	inner map[T]struct{}
}

func NewSet[T comparable](values ...T) *Set[T] {
	s := &Set[T]{
		inner: make(map[T]struct{}),
	}

	for _, v := range values {
		s.Add(v)
	}

	return s
}

func (s *Set[T]) Add(v T) {
	s.inner[v] = struct{}{}
}

func (s *Set[T]) Remove(v T) {
	delete(s.inner, v)
}

func (s *Set[T]) Contains(v T) bool {
	_, ok := s.inner[v]
	return ok
}

func (s *Set[T]) Len() int {
	return len(s.inner)
}

func (s *Set[T]) Values() []T {
	values := make([]T, 0, len(s.inner))
	for v := range s.inner {
		values = append(values, v)
	}

	return values
}
```

## 使用 gorm 查询的常用操作

```go
func QueryObjectsByCondsWithDb[T any](db *gorm.DB, conds ...interface{}) ([]T, error) {
	var objects []T
	err := db.Find(&objects, conds...).Error
	if err != nil {
		return nil, err
	}

	return objects, nil
}

func QueryObjectMapByCondsWithDb[K comparable, V any](
	db *gorm.DB,
	keyOf func(v *V) K,
	conds ...interface{},
) (map[K]*V, error) {
	var objects []V
	err := db.Find(&objects, conds...).Error
	if err != nil {
		return nil, err
	}

	objectMap := make(map[K]*V)
	for i := range objects {
		v := &objects[i]
		objectMap[keyOf(v)] = v
	}

	return objectMap, nil
}
```

## 尝试实现类似 Rust 风格的迭代器

```go
package main

import (
	"fmt"
)

type FilterFn[T any] func(v T) bool
type MapFn[T any] func(v T) T

type Filter[T any] struct {
	Apply FilterFn[T]
}

func (f Filter[T]) Next(v T) (T, bool) {
	return v, f.Apply(v)
}

type Map[T any] struct {
	Apply MapFn[T]
}

func (m Map[T]) Next(v T) (T, bool) {
	return m.Apply(v), true
}

type Take[T any] struct {
	n     int
	taken int
}

func (t *Take[T]) Next(v T) (T, bool) {
	if t.taken >= t.n {
		return v, false
	}

	t.taken += 1
	return v, true
}

type Skip[T any] struct {
	n       int
	skipped int
}

func (t *Skip[T]) Next(v T) (T, bool) {
	if t.skipped < t.n {
		t.skipped += 1
		return v, false
	}

	return v, true
}

type IteratorNext[T any] interface {
	Next(v T) (T, bool)
}

type Iterator[T any] struct {
	inner      []T
	operations []IteratorNext[T]
}

func (iter *Iterator[T]) Filter(fn FilterFn[T]) *Iterator[T] {
	iter.operations = append(iter.operations, &Filter[T]{
		Apply: fn,
	})

	return iter
}

func (iter *Iterator[T]) Map(fn MapFn[T]) *Iterator[T] {
	iter.operations = append(iter.operations, &Map[T]{
		Apply: fn,
	})

	return iter
}

func (iter *Iterator[T]) Take(n int) *Iterator[T] {
	iter.operations = append(iter.operations, &Take[T]{
		n: n,
	})

	return iter
}

func (iter *Iterator[T]) Skip(n int) *Iterator[T] {
	iter.operations = append(iter.operations, &Skip[T]{
		n: n,
	})

	return iter
}

func (iter *Iterator[T]) applyOperations(v T) (T, bool) {
	var (
		tmp  = v
		keep = false
	)
	for i := range iter.operations {
		tmp, keep = iter.operations[i].Next(tmp)
		if !keep {
			return tmp, keep
		}
	}

	return tmp, keep
}

func (iter *Iterator[T]) Collect() []T {
	var result []T

	for i := range iter.inner {
		v, keep := iter.applyOperations(iter.inner[i])
		if !keep {
			continue
		}

		result = append(result, v)
	}

	return result
}

func (iter *Iterator[T]) Count() int {
	return len(iter.Collect())
}

func IntoInterator[T any](ss []T) *Iterator[T] {
	return &Iterator[T]{
		inner: ss,
	}
}

func main() {
	var data []int
	for i := 0; i < 100; i++ {
		data = append(data, i)
	}
	data = IntoInterator(data).
		Filter(func(v int) bool { return v%2 == 0 }).
		Map(func(v int) int { return 2 * v }).
		Skip(5).
		Take(5).
		Collect()

	fmt.Printf("data: %#v\n", data) // data: []int{20, 24, 28, 32, 36}
}
```

# 对性能的影响

我们定义一个 IntSum 和一个 GenericSum：

```go
type MyTypeSet interface {
	~int | ~float32 | ~string
}

func GenericSum[T MyTypeSet](values []T) T {
	var s T
	for _, v := range values {
		s += v
	}
	return s
}

func IntSum(values []int) int {
	var s int
	for _, v := range values {
		s += v
	}
	return s
}
```

增加性能测试的代码：

```go
func BenchmarkGenericSum(b *testing.B) {
	var data []int
	for i := 0; i < 100; i++ {
		data = append(data, i)
	}

	for i := 0; i < b.N; i++ {
		_ = GenericSum(data)
	}
}

func BenchmarkIntSum(b *testing.B) {
	var data []int
	for i := 0; i < 100; i++ {
		data = append(data, i)
	}

	for i := 0; i < b.N; i++ {
		_ = IntSum(data)
	}
}
```

运行测试，获取测试结果：

```
goos: darwin
goarch: amd64
cpu: Intel(R) Core(TM) i5-7267U CPU @ 3.10GHz
BenchmarkGenericSum-4   24427492	        50.50 ns/op	       0 B/op	       0 allocs/op
BenchmarkIntSum-4   	31268010	        38.90 ns/op	       0 B/op	       0 allocs/op
```

可以看到泛型版本比普通版本慢了差不多 25%，这是为什么？

这和 Go 的泛型实现有关，具体可以参考这篇文章[编程语言是如何实现泛型的](https://www.bmpi.dev/dev/deep-in-program-language/how-to-implement-generics)。


# 参考资料

- [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)
- [All your comparable types](https://go.dev/blog/comparable)
- [编程语言是如何实现泛型的](https://www.bmpi.dev/dev/deep-in-program-language/how-to-implement-generics)
