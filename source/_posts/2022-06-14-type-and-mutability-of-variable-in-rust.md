---
title: Rust 中变量的类型和可变性
tags:
  - Rust
description: 初学 Rust 的时候，容易将可变引用类型和自身可变性弄混，下面以几个例子来说明其中的区别。
index_img: /img/rustacean-flat-gesture.png
categories:
  - Rust
date: 2022-06-14 16:09:40
---


初学 Rust 的时候，容易将可变引用类型和自身可变性弄混，下面以几个例子来说明其中的区别。

下面代码中的 `r1`、`r2` 和 `r3` 有啥区别？

```rust
let mut string1 = String::from("hello");

let r1 = &mut string1;
let mut r2 = &string1;
let mut r3 = &mut string1;
```

咋一看好像都差不多，实际上有类型和自身可变性的区别。

## 类型

这里增加一个获取类型的函数，并降每个类型打印出来:

```rust
fn type_of<T>(_: &T) -> &'static str {
    std::any::type_name::<T>()
}

fn main() {
    let mut string1 = String::from("hello");

    let r1 = &mut string1;
    println!("type of r1: {}", type_of(&r1));

    let mut r2 = &string1;
    println!("type of r2: {}", type_of(&r2));

    let mut r3 = &mut string1;
    println!("type of r3: {}", type_of(&r3));
}
```

运行得到结果：

```
type of r1: &mut alloc::string::String
type of r2: &alloc::string::String
type of r3: &mut alloc::string::String
```

这里可以看到 `r1` 和 `r3` 的类型都是 `&mut String` 而，`r2` 的类型为 `&String`。

## 可变性

在原本的基础上，我们再添加几行代码，目的是尝试给 `r1`、`r2`、`r3` 赋值。

```rust
let mut string2 = String::from("world");

r1 = &mut string2;
r2 = &string2;
r3 = &mut string2;
```

常识编译执行，将会得到下面的错误：

```
error[E0384]: cannot assign twice to immutable variable `r1`
  --> src/bin/type_and_self_mutability_of_var.rs:17:5
   |
6  |     let r1 = &mut string1;
   |         --
   |         |
   |         first assignment to `r1`
   |         help: consider making this binding mutable: `mut r1`
...
17 |     r1 = &mut string2;
   |     ^^^^^^^^^^^^^^^^^ cannot assign twice to immutable variable
```

根据报错信息，我们得知不能修改不可变变量 `r1` 的值，虽然 `r1` 的类型是 `&mut String`。
注释掉 `r1 = &mut string2;` 后就能正常编译了。三个变量情况汇总成表格：

| 变量 | 类型 | 是否可变 |
| :-: | :-: | :-: |
| r1 | `&mut String` | 否 |
| r2 | `&String` | 是 |
| r3 | `&mut String` | 是 |

可以通过 `mut` 关键字使得变量自身可变，比如上面代码中的：

```rust
let mut r2 = &string1;
let mut r3 = &mut string1;
```

`mut` 关键字还能在函数声明参数地方和模式匹配的时候使用，比如：

```rust
fn foo(mut bar: String) {
    // 可以在里面修改 bar
    ...
}
```

和

```rust
if let Some(mut proxy_channel) = proxy_channel_rx.recv().await {
    // 修改 proxy_channel
    ...
}
```



## 结论

- 类型为 `&mut T` 不代表可以修改自身
- 使用 `mut` 关键字使变量自身可变
