---
title: constexpr
date: 2025-10-10 14:43:24
categories: cpp
---

# 概念

常量表达式是C++11的关键字，在编译时就可以计算表达式的值

> 作用：
>
> 1. 编译时计算
> 2. 性能优化
> 3. 类型安全

和const的区别是其为编译时初始化，const为运行时初始化。

# 修饰变量

编译期常量

# 修饰函数

函数在编译期即可调用：

```cpp
constexpr int square(int x){
    return x*x;
}

constexpr int val = square(5);
```

