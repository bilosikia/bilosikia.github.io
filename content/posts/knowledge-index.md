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

- n & -n returns the **rightmost 1 bit in n**.

- n & (n - 1) 消除最右 1

- GUARDED_BY

- 容器元素比较：严格弱序

- s.size() - 10 的结果是无符号

- COW: [std::string的Copy-on-Write：不如想象中美好](https://www.cnblogs.com/promise6522/archive/2012/03/22/2412686.html)

- [伪共享（false sharing），并发编程无声的性能杀手](https://www.cnblogs.com/cyfonly/p/5800758.html)

- 17 版本后，可以直接写 std::array vec = {"hello"}, 不用写类型和个数了

- 对 cast 完的指针就行 dereference 是 UB(TBAA 优化)

  ![image-20210720164420645](/Users/gaokuilin/Library/Application Support/typora-user-images/image-20210720164420645.png)

## template 

- 测试类型
- 测试满足条件
- static_assert 不能直接写 false
- [协变&逆变](https://www.jianshu.com/p/db76a8b08694)

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
  
- [fat pointer](https://guihao-liang.github.io/2020/06/06/fat-pointer) 

- [Higher-Rank Trait Bounds](https://zhuanlan.zhihu.com/p/404574814)

- [generator 内存优化](https://tmandry.gitlab.io/blog/posts/optimizing-await-1/)

# 网络

- [layer 4负载均衡](https://www.nginx.com/resources/glossary/layer-4-load-balancing/)
- [https详解](https://segmentfault.com/a/1190000021494676)
- [TLS安全网络传输协议简介-侯涛](https://www.bilibili.com/video/BV184411777S)

