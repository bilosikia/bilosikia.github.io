---
title: "rust Pin Project"
date: 2022-12-29T17:52:54+08:00
toc: true
---

# 再谈 pin 语义

在之前，我一直以为 pin 的作用仅仅是通过无法从 Pin<P<T>> 获得类似 &mut T的指针，从而能确保 T 不会 move。 一旦 Pin<P<T>> 离开作用域被 drop 时，T 就不在受 Pin 的约束了，语法上就可以被 move。

但在后来思考为什么 [map_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked) 是 unsafe 时，才发现之前对 Pin 的语义理解存在问题。 [map_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked) 的 Self 类型等同于 Pin<&T>，通过一个 FnOnce(&T) -> &U 将 &T 映射为 &U，不管是 Pin<&T> 还是 FnOnce(&T) -> &U，都仅仅只是 &，都没有 move T 的可能，为什么该函数却是 unsafe 的呢？ 在重新阅读了 pin 的文档后，有一个很重要的点被我遗漏了：

> At a high level, a `Pin``<P>` ensures that the pointee of any pointer type `P` has a stable location in memory, meaning it cannot be moved elsewhere and its memory cannot be deallocated until **it gets dropped.**

简而言之，只要被 pin 过，不管 Pin<P<T>> 是否依然有效，pin 语义会一直保持到 T 被 dropped。 在 [new_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked) 的示例中，也给了一个示例:

```Rust
fn move_pinned_ref<T>(mut a: T, mut b: T) {
    unsafe {
        let p: Pin<&mut T> = Pin::new_unchecked(&mut a);
        // This should mean the pointee `a` can never move again.
    }
    mem::swap(&mut a, &mut b);
    // The address of `a` changed to `b`'s stack slot, so `a` got moved even
    // though we have previously pinned it! We have violated the pinning API contract.
}
```

这个语义其实非常的特殊，Pin<P<T>> 被析构后，但它的作用却依然还生效。那如何确保没有 Pin<P<T>>  时，P<T> 不会违背 Pin 的保证呢？这就是为什么 [new_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked) 和 [map_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked) 是 unsafe 的原因。通过 unsafe，将 pin 之后的 T 不会 move 的责任转移到开发者，由开发者在构造 Pin<P<T>> 时承诺，我决不会 move 直到 T drop。 [map_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked) 为什么必须是 unsafe 可以参考这个[回答](https://stackoverflow.com/questions/74908088/why-rust-pin-map-unchecked-is-unsafe)。

为什么语言设计上要设计成这样呢，我个人认为，需要使用到 Pin，则说明该对象不能被 move。如果设计成只有持有 Pin 时，对象才无法被移动，在没有 Pin 时，该对象被移动，将会导致 UB，但是却没有违背任何约束。

# 什么是 pin project

首先，pin 是非递归的。当一个结构被 pin 住时，只是表示这个结构整体不能移动，结构的某些字段是否能够移动 和当前结构是否被 pin 住完全没有关系。这样设计的好处是，更够进行更细粒度的控制，如果结构的某些字段不影响结构整体被 pin 住的语义，那就可以从 Pin<P<T>> 中获得该字段的 mut 引用，反之，如果某些字段应该随着结构被 pin 住，则不应该从 Pin<P<T>> 中获得该字段的 mut 引用。

- 从 Pin<P<T>> 获得 Pin<&mut Field>, 称为 **structural pinning**。
- 从 Pin<P<T>> 获得 &mut Field, 称为 **non-structural**。

一个字段应该是 **structural** 还是 **non-structural**，需要根据是否有不变式需要被保证。rust 中所有对象都是 move 的，即便该对象没有实现 Unpin, 但一旦该对象被 pin 住了，之后就不能被移动了，需要由开发者自己来保证这个不变式。如果一个对象在 pin 住之前移动，是没有破坏任何不变式的，虽然可能会导致 UB。正确设计一个字段是否是**structural** 是开发者的责任，pin 只是一个工具，如果设计的不当，一些需要被保证的不变式将会被破坏。

当 **structural pinning** 某个字段时，有些额外的要求：

- 只有在 T 的所有字段都是 Unpin 时，才能为 T 实现 Unpin。反证法，如果给 T 实现 Unpin，不需要 unsafe 就可以获得 &mut T, 然后就可以获得 &mut Field, 违背了 Field 可能已经被 pin 住的不变式了。 Unpin 是一个 safe auto trait，默认是 T 的所有字段都是 Unpin 时，自动为 T 实现 Upin，但也可以手动为 T 实现 Unpin，且不需要 unsafe。
- T 的 Drop 实现，不应该移动 T。由于历史原因，drop 的参数为 &mut T，这就给了移动 T 的机会，需要由开发者保证不会在 Drop 实现中，移动 T，详见 pin 文档的 [Drop implementation](https://doc.rust-lang.org/std/pin/index.html#drop-implementation) 模块。
- 需要确保 T 的 [Drop guarantee](https://doc.rust-lang.org/std/pin/index.html#drop-guarantee)。对于被 pin 住对象，在析构函数调用之前，任何使得被 pin 住对象所在的内存 invalidated，repurposed，deallocated 的操作都是不允许的。
- 不能提供任何导致该字段被 move 的操作。比如提供 fn(Pin<&mut T<Field>>) -> Option<Field> 此类导致 Field 被移动的方法，或者提供返回 &mut Field 的方法。
- \#[repr(packed)] 类型的结构都不能被 pin，更不用说 **structural** 了。

除了第一条，剩下的都是 Pin<P<T>> 时，T 本身需要需要遵循的规则。当 **structural** **pinning** 时，表示 T 的一部分内存一定需要被 pin 住，任何通过 &mut T 移动这部分内存的操作都是不允许的。

# pin-project 的安全性保证

在和 pin 打交道时，特别是 pin project（[map_unchecked](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked)，[map_unchecked_mut](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked_mut) 都是 unsafe，这也是为什么 Unpin 是 safe  tarit 的原因）时，总是绕不开 unsafe。[pin_project](https://docs.rs/pin-project/latest/pin_project/attr.pin_project.html) 通过提供的一些宏，使得在和 pin 打交道时，可以完全不使用任何 unsafe，详见 [Safety](https://docs.rs/pin-project/latest/pin_project/attr.pin_project.html#safety) 模块。

简单来说，pin-project 通过生成的代码，使得开发者没有机会破坏 pin 语义：

- 通过 blanket impl 的方式，无法手动为 T 实现 Unpin，确保了仅仅在所有字段都实现 Unpin 时，T 才会 Unpin。
- 通过占位实现 Drop 的方式，无法直接为 T 实现 Drop，要自定义 drop 只能通过`#[pinned_drop]`，而参数只能是 Pin<&mut Self>。
- 通过定义 assert 辅助函数，确保无法为 #[repr(packed)] 添加 #[pin_project] 宏，详见[这儿](https://github.com/taiki-e/pin-project/pull/34)。
- 而其他的约束，比如 [Drop guarantee](https://doc.rust-lang.org/std/pin/index.html#drop-guarantee)，通过 Pin<P<T>> 移动，本身都需要 unfase，仅使用 safe 代码是无法违背这些约束的。

值得注意的是，使用 pin-project 时，如果没有字段使用 #[pin], 即便结构中有实现 !Unpin 的字段，pin-project 也会结构实现 Unpin。

```Rust
use std::marker::PhantomPinned;
use pin_project::pin_project;

#[pin_project]struct Struct<T> {
    field: T,
    #[pin] // <------ This `#[pin]` is required to make `Struct` to `!Unpin`.
    _pin: PhantomPinned,
}
// Note that using PhantomPinned without #[pin] attribute has no effect.
```

当所有字段都没有使用 #[pin]，则意味着所有字段都可以从 Pin<P<T>> 获得 &mut Field，则相当于整个结构没有被 Pin 住，故 pin-project 会为 T 实现 Unpin。这和 Unpin 的自动实现规则不同，使用 pin-project 时，我们需要手动通过 #[pin] 指定哪些字段需要 **structural**，如果不指定，默认为 **non-structural。**

pin-project 的文档对理解 pin 有巨大的帮助，想要深入理解 pin，熟悉 pin-project 是必不可少的。
