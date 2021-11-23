

## Drop

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