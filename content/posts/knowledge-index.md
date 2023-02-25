---
title: "knowledge index"
date: 2021-06-09T14:55:59+08:00
---

# Linux

- [再谈 slab](https://zhuanlan.zhihu.com/p/61457076)
- 伙伴算法
- [Socket write & read](https://www.cnblogs.com/junneyang/p/6126635.html)
- [seq锁](https://zhuanlan.zhihu.com/p/364044850)
- [惊群效应](https://blog.csdn.net/lyztyycode/article/details/78648798)
- [BBR](https://www.bilibili.com/video/BV1h64y1x72H?spm_id_from=333.999.0.0)
- [cache](http://www.wowotech.net/memory_management/458.html)

# c++

- 取二进制最右非 0 位：n & (~(n - 1)), 但是有溢出的风险

- n & -n returns the **rightmost 1 bit in n**.

- n & (n - 1) 消除最右 1

- X % 2^n = X & (2^n - 1)，环形队列，时间轮实现

- GUARDED_BY

- 容器元素比较：严格弱序

- s.size() - 10 的结果是无符号

- [intrusive](https://stackoverflow.com/questions/5004162/what-does-it-mean-for-a-data-structure-to-be-intrusive)

- COW: [std::string的Copy-on-Write：不如想象中美好](https://www.cnblogs.com/promise6522/archive/2012/03/22/2412686.html)

- [伪共享（false sharing），并发编程无声的性能杀手](https://www.cnblogs.com/cyfonly/p/5800758.html)

- 17 版本后，可以直接写 std::array vec = {"hello"}, 不用写类型和个数了

- 对 cast 完的指针就行 dereference 是 UB(TBAA 优化)

- [cmake变量](https://mp.weixin.qq.com/s?__biz=MzU1OTgxNDY0Ng==&mid=2247483723&idx=1&sn=c9569b2ba072a6c32c231199a18be699&chksm=fc10c3d2cb674ac4c44f7c834b4a238e0f95363b99d206650bf161be749cebf1ba08c0da7354&token=2028512086&lang=zh_CN#rd)

- [Why Memory Barriers](http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html)

- [jemalloc](https://mp.weixin.qq.com/s/fB0IILuBBuk_YrKNjW_3Ag)

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

- [Dynamic Dispatch in Rust](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/)

- [Higher-Rank Trait Bounds](https://zhuanlan.zhihu.com/p/404574814)

- [generator 内存优化](https://tmandry.gitlab.io/blog/posts/optimizing-await-1/)

- Trait 不仅可以包含 type 和 function，还可以包含常量

- [&'static T vs T: 'static](https://doc.rust-lang.org/rust-by-example/scope/lifetime/static_lifetime.html)

- MutexGuard !send but sync

- [[slice](https://doc.rust-lang.org/std/slice/index.html)::[from_ref](https://doc.rust-lang.org/std/slice/fn.from_ref.html#)]

# 网络

- [layer 4负载均衡](https://www.nginx.com/resources/glossary/layer-4-load-balancing/)
- [https详解](https://segmentfault.com/a/1190000021494676)
- [TLS安全网络传输协议简介-侯涛](https://www.bilibili.com/video/BV184411777S)

# go

- [package和文件夹的关系](https://www.zhihu.com/question/60426831)
- [Go包初始化顺序](https://zhuanlan.zhihu.com/p/183044306)



# 分布式

- [分布式系统一致性的发展历史](https://danielw.cn/history-of-distributed-systems-1)
- 

# 其他

- [np问题](http://www.matrix67.com/blog/)
- [大佬的总结](https://tanxinyu.work/2021-annual-summary/)
  - 一位清华大佬的年终总结，总结了本科到研三的学习经历，一些值得学习的经验思考。能从文章中看到优秀的人是如何学习，如何安排自己时间，如果规划工作的，总结部分能引起强烈共鸣。
