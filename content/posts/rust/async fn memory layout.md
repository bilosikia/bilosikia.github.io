---
title: "async fn memory layout"
date: 2021-11-23T14:55:59+08:00
toc: true
---

这个问题源于 rust 的一个 issue: [Async fn doubles argument size](https://github.com/rust-lang/rust/issues/62958)

考虑下面的代码，可能会产生一下几个问题：

- async fn 生成的对象内存如何布局？

- 为什么 fut1 和 fut2 的大小不一样?

```Rust
async fn wait() {}

async fn foo(arg: [u8; 10]) {
    wait().await;
    drop(arg);
}

fn main() {
    let fut1 = async {
        let arg = [0u8; 10];
        wait().await;
        drop(arg);
    };
    let fut2 = foo([0u8; 10]);
    println!("{}, {}", std::mem::size_of_val(&fut1), std::mem::size_of_val(&fut2));
}
12, 22
```

## async fn 对象内存布局

每个 async 函数被编译器实现为一个 generator，每个 await 对应于一个 yield ，每个 yield 点需要保存当前必要的上下文用于后续恢复执行。

```rust
let xs = vec![1, 2, 3];
let mut gen = || {
let mut sum = 0;
    for x in xs.iter() {  // iter0
        sum += x;
        yield sum;  // Suspend0
    }
    for x in xs.iter().rev() {  // iter1
        sum -= x;
        yield sum;  // Suspend1
    }
};

//  maybe?
enum SumGenerator {
    Unresumed { xs: Vec<i32> },
    Suspend0 { xs: Vec<i32>, iter0: Iter<'self, i32>, sum: i32 },
    Suspend1 { xs: Vec<i32>, iter1: Iter<'self, i32>, sum: i32 },
    Returned
}
```

rust 的 enum 为 tag enum，理论上内存占用为 ：tag + 最大内存占用 variant + 对齐。

对 async 函数，执行到一个新的 await 点时，即转变为另一个 variant，由于 enum 复用一块内存，内存被重新布局，可能发生 move。

但 future 对象可能包含自引用，move 后，内存安全将不被保证。所以 async 函数的每一个 variant，都有自己的内存，也就导致了最开始的问题。

实际上，生成器的对象布局是编译器内部行为，一般只是以 Enum 的方式来表述状态机，在实现上，并不一定是 Enum 的方式。

## fut1 和 fut2 的区别

async 函数生成的代码，将参数作为了 Unresumed 的一个状态值保存了。

```rust
enum Fut1Generator {
    Unresumed,
    Suspend0 { arg: [u8; 10] },
    Returned
}

enum Fut2Generator {
    Unresumed { arg: [u8; 10] }, // future delay 执行者， arg 最为函数参数被捕获
    Suspend0 { arg: [u8; 10] },
    Returned
}
```



## reference

https://tmandry.gitlab.io/blog/posts/optimizing-await-2/

介绍了 rust 的一些生成器内存布局的优化，但只介绍了 local 变量的优化