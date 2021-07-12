---
title: "Coredump"
date: 2021-06-27T15:40:32+08:00
draft: true
---

# 产生 coredum

1. ulimit -c unlimited 取消 coredump 文件大小限制
2. sudo sysctl kern.coredump=1 mac 下，开启系统 coredump 选项
3. 另外可能需要设置 mac 下 /cores/ 目录写权限

# 分析

