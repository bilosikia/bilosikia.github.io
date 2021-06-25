---
title: "知识索引"
date: 2021-06-09T14:55:59+08:00
draft: false
---

# Linux

- [再谈 slab](https://zhuanlan.zhihu.com/p/61457076)
- 伙伴算法
- [Socket write & read](https://www.cnblogs.com/junneyang/p/6126635.html)

# c++

- 取二进制最右非 0 位：n & (~(n - 1)), 但是有溢出的风险
- GUARDED_BY
- 容器元素比较：严格弱序

# Rust

- [dyn Trait and impl Trait in Rust](https://www.ncameron.org/blog/dyn-trait-and-impl-trait-in-rust/)

- [On Generics and Associated Types](https://blog.thomasheartman.com/posts/on-generics-and-associated-types)

- inclusive ranges

  ```rust
  fn main() {
      for i in 0..=26 {
          println!("{}", i);
      }
  }
  ```

- matches!

  ```
  let bar = Some(4);
  assert!(matches!(bar, Some(x) if x > 2));
  ```
