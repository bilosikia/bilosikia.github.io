---
title: "lifetime 小问"
date: 2021-11-23T14:55:59+08:00
toc: true
---

#  非词法作用域

Rust 的生命周期检查为静态检查, 并且为非词法作用域。

*Rust把生命周期检查的步骤由HIR改为了MIR，以便可以降低生命周期检查的粒度，使生命周期规则从词法作用域转变成非词法作用域*

```rust
let mut data = vec![1, 2, 3];
let x = &data[0];
println!("{}", x);
// This is OK, x is no longer needed
data.push(4);
```

但如果 x 实现了 Drop 会怎么样呢？

```rust
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

let mut data = vec![1, 2, 3];
let x = X(&data[0]);
println!("{:?}", x);
data.push(4);
// Here, the destructor is run and therefore this'll fail to compile.
```

当实现 Drop 时，生成的代码会隐式的添加 drop 调用，实际的生命周期比看到的非词法作用域要长。

# static

static lifetime 和 static bound 是不同的概念。

`&'static` 表示被引用的对象具有 static 生命周期，被引用对象必须要活得跟剩下的程序一样久。描述的是被引用对象，而不是引用自身。至于对象是什么时候创建的并不重要，可以是编译时的 static 对象，也可以是动态创建的：

~~~rust
// 字符串常量
let str_literal: &'static str = "str literal";
// 静态全局变量
static mut MUT_BYTES: [u8; 3] = [1, 2, 3];
// 运行时创建的对象，但生命周期和整个程序一样长
fn rand_str_generator() -> &'static str {
    let rand_string = rand::random::<u64>().to_string();
    Box::leak(rand_string.into_boxed_str())
}

fn static_life(t: &'static str) {}
static_life("123");
// 编译报错：argument requires that borrow lasts for `'static`
// static_life(&String::from("123"));
~~~

`T: 'static` 表示的是 T 被 static bound 约束，T 持有的对象可以一直到程序结束，但并不是必须活的和程序一样久。String，Vec 等 owner 对象满足 static bound，因为持有这种对象无限长是安全的，他们内部没有引用其他对象。如果 T
内部包含了其他对象的引用，能够被无限持有还需要看内部被引用对象的生命周期。

```rust
fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}
// owned 的 String
drop_static(String::from("hello"));

fn static_bound<T: 'static>(t: &T) {}
static_bound(&String::from("123"));
static_bound(&Box::new("123"));
let s = String::from("123");
// 编译报错：argument requires that `s` is borrowed for `'static`
// static_bound(&Box::new(&s));
```

`&'static T` 和 `T: 'static` 的关系：

- `&'static T` 满足 static bound。
- 满足 static bound 的不一定需要是 static 生命周期

# 生命周期案例分析

```Rust
struct Text<'txt> {
    content: &'txt str
}
struct RefText<'txt> {
    ptr: &'txt mut Text<'txt>
}
fn main() {
    let mut txt = Text {
        content: "eeee",
    };
    {
        let r = RefText {
            ptr: &mut txt,
        };
    }
    use_text(&mut txt);
}
fn use_text(_list: &mut Text) {

}

error[E0499]: cannot borrow `txt` as mutable more than once at a time
  --> src/main.rs:19:14
   |
16 |             ptr: &mut txt,
   |                  -------- first mutable borrow occurs here
...
19 |     use_text(&mut txt);
   |              ^^^^^^^^
   |              |
   |              second mutable borrow occurs here
   |              first borrow later used here
```

将生命周期展开：

```rust
'a: {
    let content: &'a str = "eeee";
    let mut txt: Text<'a> = Text<'a> {
        content: content,
    };
    'b: {
        // 由于 RefText 定义的限制，这里 ptr 的生命周期不能是 'b
        // let ptr: &'b mut Text<'a> = &'b mut txt;
        let ptr: &'a mut Text<'a> = &'a mut txt;
        let r: RefText<'a> = RefText<'a> {
            ptr: ptr,
        };
    }
    // a 的生命周期内，已经被借用了
    'c: {
        use_text(&'c mut txt);
    }
}
```

如果是如下的形式，将不存在上面的问题：

```rust
struct RefText<'ptr, 'txt> {
    ptr: &'ptr mut Text<'txt>
}
```

推导 `ptr` 类型时，为什么不能是 `&'b mut Text<'b>`? 涉及到协变和逆变问题：

对于可变引用`&'a mut T`，对 `'a` 是可以协变的，即你用一个比 `'a` 活得久的赋值给它是 OK 的，但是 `T` 是不可形变的。如上面的 `ptr`，它对 `T` 为 `Text<'a>` 是不可形变的，而 `ptr` 自身的生命周期可以协变，所以 `ptr` 可以是 `&'b mut Text<'a>` 或 `&'a mut Text<'a>`，而不能是 `&'b mut Text<'b>`。

# Reference

- https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program

- https://doc.rust-lang.org/nomicon/subtyping.html