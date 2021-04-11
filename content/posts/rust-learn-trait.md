---
title: "Rust学习——trait"
date: 2021-04-10T17:30:17+08:00
draft: false
---

trait 是 Rust 的灵魂。从语义上来说，trait 是在行为上对类型的约束，这种约束可以让 trait 有如下四种用法：

- 接口抽象：接口是对类型行为的统一约束；
- 泛型约束：泛型的行为被 trait 限定在更有限的范围内；
- 抽象类型：在运行时可以作为一种间接的抽象类型去使用，动态的分发给具体的类型，但是有一定的性能开销；
- 标签 trait：对类型的约束，可以直接作为一种“标签”使用。

## 接口抽象

trait 最基础的用法就是进行接口抽象，可以看做是其他语言中的 interface，例如 golang 中的 interface 定义了一些行为：

```go
type Cat interface {
    Miao() string
    Parkour() string
}
```

```rust
// rust 中的 trait
pub trait Cat {
    fn miao(&self) -> String;
    fn parkour(&self) -> String;
}
```

看起来是不是如出一辙，trait 告诉 Rust 编译器某个特定类型拥有可能与其他类型共享的功能。可以通过 trait 以一种抽象的方式定义共享的行为。其特点如下：

- 接口中可以定义方法，并且支持默认实现；
- 接口中不能实现另一个接口，但是接口之间可以继承；
- 如果要实现某个 trait，那么该 trait 和要实现该 trait 的那个类型至少有一个要在当前 crate 中定义。

### trait 继承

Rust 不支持传统面向对象的继承，但是支持 trait 继承。子 trait 可以继承父 trait 中定义或实现的方法。例如：

``` rust
trait People {
    fn speak();
    fn walk();
}
trait Teacher: People {
    fn teach_student();
}
```

需要注意的是包含 trait 限定的泛型属于静态分发，所以调用 trait 限定中的方法也都是零成本。

## 泛型约束

在使用泛型时，如果想限定能够使用该泛型行为的对象时，就可以给改泛型加上 trait 约束。

``` rust
use std::ops::Add;
fn sum<T: Add<T, Output=T>>(a: T, b: T) -> T {
    a + b
}

fn main() {
    assert_eq!(sum(1u32, 2u32), 3);
}
```

## 抽象类型

trait 还可以用来当作抽象类型，抽象类型是类型系统的一种，也叫做存在类型。相对于具体类型而言，抽象类型无法直接实例化，他的每个实例都是具体类型的实例。

编译器可能无法确定抽象类型的确切功能和所占空间大小，所以 Rust 目前有两种方法来处理抽象类型：▪`trait对象`▪ 和 ▪`impl Trait`▪。

### trait 对象

trait 对象就是拥有相同行为的类型的抽象对象，也称为 `Trait Object`。需要记住的是，用 trait 限定的泛型参数是属于静态分发，而使用 trait object 属于动态分发，两者在性能上是有差别的。

```rust
#[derive(Debug)]
struct Foo;

trait Bar {
    fn baz(&self);
}

impl Bar for Foo {
    fn baz(&self) {
        println!("{:?}", self)
    }
}

fn static_dispatch<T>(t: &T) where T: Bar {
    t.baz();
}

fn dynamic_dispatch(t: &dyn Bar) {
    t.baz();
}


fn main() {
    let foo = Foo;
    static_dispatch(&foo);
    dynamic_dispatch(&foo);
}
```

trait 本身就是一种抽象而非具体类型，所有他的类型在编译期是不确定的，所以 trait 对象需要使用胖指针，可以利用&或者 Box 来指针一个 trait 对象。Trait Object 包括两个指针：data 指针和 vtable 指针。vtable包含了 Trait Object 的方法、析构函数、大小和对齐等信息。

![img](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/traitobject.png)

trait object 会根据虚表指针中查出正确的函数指针，然后再进行动态调用。

## 标签 trait

利用 trait 的特性，我们也可以用 trait 来给类型加上标签，Rust 提供了 5 个重要的标签 trait，都被定义在标准库 std::maker 模块中。

它们分别是：

- Sized trait：用来标识编译期可确定大小的类型。
- Unsize trait：用于标识动态大小类型（DST）。
- Copy trait：用来标识可以安全按位复制其值的类型。
- Send trait：用来标识可以跨线程安全通信的类型。
- Sync trait：用来标识可以在线程间安全共享引用的类型