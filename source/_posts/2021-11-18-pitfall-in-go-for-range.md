---
title: Go - for range 的陷阱
date: 2021-11-18 20:49:55
description: for range 是 Go 中最常用的语句之一，但是新手可能不知道在 Go 的 for range 里的中循环变量是共享的。这可能会引起一些问题。
index_img: img/gopher.png
tags: 
    - Go
categories:
    - Go
---

Go 的 `for range` 里的**循环变量是共享的**，这可能会引起一些问题，以下面的例子为例：


```go
type Item struct {
	ID     int64
	Name   string
	Amount int
}

items := []Item{
    {
        ID: 1,
        Name: "mushroom",
        Amount: 2,
    },
    {
        ID: 2,
        Name: "apple",
        Amount: 5,
    },
    {
        ID: 3,
        Name: "ore",
        Amount: 1,
    },
}

itemMap := make(map[int64]*Item)
for _, item := range items {
    itemMap[item.ID] = &item
}

for id, item := range itemMap {
    fmt.Printf("id: %d item: %#v\n", id, *item)
}
```

上面的代码大致的意思是基于 `items` 创建一个 `id` 到 `*Item` 的映射，执行后结果为：

```
id: 1 item: main.Item{ID:3, Name:"ore", Amount:1}
id: 2 item: main.Item{ID:3, Name:"ore", Amount:1}
id: 3 item: main.Item{ID:3, Name:"ore", Amount:1}
```

可以看到结果数据和预期不符，主要的原因就是因为**循环变量是共享的**，我们可以打印循环变量的地址来验证：

```go
for _, item := range items {
    fmt.Printf("%p\n", &item)
}
```

结果输出：

```
0xc000124000
0xc000124000
0xc000124000
```

可以看到 `item` 的地址一直是 `0xc000124000`，故可以明白为什么上面的 `map` 的结果不符合预期。

要避免这种情况，只需要记住：**用 for range 在遍历的时候，避免取循环变量的地址作为结果，除非你知道自己在干什么**

上面的需求可以通过另一种形式取值来达成：

```go
itemMap := make(map[int64]*Item)
for offset := range items {
    itemMap[items[offset].ID] = &items[offset]
}

for id, item := range itemMap {
    fmt.Printf("id: %d item: %#v\n", id, *item)
}
```

最终结果：

```
id: 1 item: main.Item{ID:1, Name:"mushroom", Amount:2}
id: 2 item: main.Item{ID:2, Name:"apple", Amount:5}
id: 3 item: main.Item{ID:3, Name:"ore", Amount:1}
```
