---
title: "rust Pin & Unpin"
date: 2021-04-30T17:52:54+08:00
---

## 如何在 Rust 中实现一个自引用的数据结构

```rust
struct SelfRef<'a> {
    s: String,
    s_ref: Option<&'a mut str>
}

fn main() {
    let mut a = SelfRef {
        s: "hello".to_owned(),
        s_ref: None
    };
    a.s_ref = Some(a.s.as_mut());
    println!("{:?}", a.s_ref);
    
    // 下面代码编译错误
    // let b = a;
    //  println!("{:?}", b.s_ref);
}

output： Some("hello")
```
代码能正常运行，a.s_ref 保存了一个指向 a.s 的引用。
但是当我们执行被注释的代码时，即尝试将 a 赋值给 b 时，编译器报了如下错误：

```rust
a.s_ref = Some(a.s.as_str());
   |                    --- borrow of `a.s` occurs here
move out of `a` occurs here, borrow later used here
```

由于 SelfRef 没有自动实现 Copy， 当执行 let b = a 时，实际发生了 move：

1. s_ref 引用指向的还是 move 前的地址，move 后，出现了悬空引用。
2.   a.s_ref 的生命周期和 a 的生命周期一样，当移动 a 时，a.s_ref 相当于还持有 a 的引用，无法移动。
3. 即便在 let b = a 之前将 a.s_ref = None，依然无法编译通过，因为 rust 的生命周期是编译时的静态检查。

由此可见，通过 & 方式的自引用结构，由于生命周期的检查，无法执行移动操作。

另外, a.s_ref 将始终持有对 a 的不可变引用（即便是 &mut str ）到 a 的生命周期结束，将无法获取 a 的可变引用。无法通过 &mut 的方式传递参数，也不能调用参数为 &mut 的方法。

我们能够构造自引用的数据类型，但是不通过 unsafe，我们能执行的操作是非常有限的，大大的限制了自引用结构的使用场景。
但是稍后将看到，自引用的结构是有必要的，我们需要一种机制来保证，我们无法移动该对象，从而确保引用的完整性。

## 通过原始指针实现自引用

除了通过引用的方式实现自引用，还可以通过指针来实现。

```rust
use std::ptr::null;

struct SelfRef {
    s: String,
    s_ref: *const String
}

fn main() {
    let mut a = SelfRef {
        s: "hello world".to_owned(),
        s_ref: null(),
    };
    a.s_ref = &a.s;
    let b = a;
    unsafe {
        println!("{:?}", *b.s_ref); // what happen
    }
}
```

原始指针不受生命周期检查的约束，所以能够编译。在我的测试中，将会打印 "hello world"。
那能说明，移动是安全的吗？并不是，对 b.s_ref 的访问依然是未定义行为，依赖于编译器是怎么实现的。
由于 a 是栈变量，b.s_ref 依然指向该栈变量一个偏移地址，a 被移动后，该变量的内存或许并没有被重写（性能？），导致看上去能正常运行。

指向自己的指针，在移动时，编译器并不会直接报错，需要一种方式来确保对象无法被移动。

## Rust 中的move

- Rust 中所有类型都是可以 move 的，包括 Pin 本身。
- 一些指针类型可以替换或者移动背后所指的对象：
  - &mut T 通过 std::mem::swap 可以交换所指的对象
  - &mut T 可以通过 std::mem::replace 替换所指对象
  -  Box<T> 可以被转换为 &mut T
  - Box<T> 可以通过 Deref 移动包含的对象

## 异步代码中的自引用

```RUST
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

async 代码将被编译器翻译成一个实现了Future的结构：

```RUST
struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf
    state: State,
}

enum State { 
    Start, 
    AwaitingReadIntoBuf, 
    Done
}

struct ReadIntoBuf {
    buf: &'a mut [u8] // 可能被实现为原始指针，这儿只是为了方便描述
}
```

如果有跨 await 的变量，该变量将在子 future 中被引用，即发生了自引用。
为了防止该 future 对象被 runtime poll 时发生移动，使用了 Pin。

```RUST
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

## Pin & Unpin

```RUST
pub struct Pin<P> {
    pointer: P,
}

pub auto trait Unpin {}

pub struct PhantomPinned;
impl !Unpin for PhantomPinned {}
```

-  Pin<P<T>> 是一个 struct 类型，保存一个指针，能确保指针所指对象T，无法被移动, 即无法直接获取保存的指针，除非 unsafe。
-  Unpin 是一个自动 Trait，实现了该 Trait 的对象，即便被包装成 Pin<P<T>>，也能安全的被移动。即实现 Unpin Trait 的类型，不受Pin 的约束, 可以直接获得Pin包含的指针，包括 &mut T，Box<T> 等。
- 编译器默认所有类型都是可以移动的，即默认为所有类型实现 Unpin。
-  !Unpin 对 Unpin 取反，!Unpin 的双重否定就是 pin， 表明该类型不能随便移动。注意， !是一个操作符, 类似的还有?。
- PhantomPinned 是一个 marker 类型，没有实现 Unpin, 用于取消编译器默认为类型实现 Unpin。

```rust
// SelfReferential 不现实Unpin
struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}
```

## Pin 的实现原理

```RUST
impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    fn deref(&self) -> &P::Target {
        Pin::get_ref(Pin::as_ref(self))
    }
}

impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```

- Pin 实现了 Deref, 即所有类型都可以获得不可变引用。
-  但只为 Target 为 Unpin 的类型实现了 DerefMut，即没有实现 Unpin 的类型不可用直接获得可变引用，也就无法移动该对象了。

```RUST
impl<P: Deref<Target: Unpin>> Pin<P> {
    pub const fn new(pointer: P) -> Pin<P> {
        unsafe { Pin::new_unchecked(pointer) }
    }

    pub const fn into_inner(pin: Pin<P>) -> P {
        pin.pointer
    }
}

impl<P: Deref> Pin<P> {
    pub fn as_ref(&self) -> Pin<&P::Target> {
        unsafe { Pin::new_unchecked(&*self.pointer) }
    }
    
    pub const unsafe fn new_unchecked(pointer: P) -> Pin<P> {
        Pin { pointer }
    }

    pub const unsafe fn into_inner_unchecked(pin: Pin<P>) -> P {
        pin.pointer
    }
}
```

- into_inner，由于实现 Unpin 的类型是可以安全 move 的，所有可以直接获取到内部指针。
- into_inner_unchecked，没有实现 Unpin 的类型，获得内部指针是 unsafe 的。 如 P 是 &mut, 需要避免在 unsafe 代码中调用 swap 和 replace。
- new_unchecked，没有实现 Unpin 的类型，通过该方法创建 Pin 是 unsafe 的，因为创建时，没有办法保证该指针所指的对象是没有被移动的（在 pin 之前），即需要用户保证数据是有效的。

```RUST
impl<P: DerefMut> Pin<P> {
    pub fn as_mut(&mut self) -> Pin<&mut P::Target> {
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }

    pub fn set(&mut self, value: P::Target)
    where P::Target: Sized {
        *(self.pointer) = value;
    }
}
```

- 提供了修改 Pin 所指对象值的方法，被指向的对象可以直接给覆盖，但是依然无法移动该对象。



## 参考：

- [Pinning - Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html)
- https://doc.rust-lang.org/std/pin/index.html
- https://www.reddit.com/r/rust/comments/n0vajw/why_the_selfreference_struct_need_pin/
