---
title: gdb常用命令
date: 2025-10-13 15:28:35
categories: linux
---

顺口溜：冷若冰霜驱动盘

```bash
l list # 查看源码

r run # 运行程序

b break # 设置断点

s step # 单步执行（进入函数）

q quit # 退出gdb

d delete # 删除断点

p print # 打印变量值
```

再加几个常用的：

```bash
bt backtrace # 查看调用栈

c continue # 继续执行

n next # 单步执行（不进入函数）

f finish # 执行完当前函数

disable 2 # 禁用编号为2的断点
```

