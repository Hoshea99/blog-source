---
title: decltype
date: 2025-10-10 11:33:37
categories: cpp
---

# 基本语法

```cpp
decltype(expression) varible;
```

- expression是一个未加括号的标识符

  直接返回该标识符的声明类型

- expression是一个函数调用或者加括号的表达式

  返回函数返回类型或者表达式结果类型（左值引用）

# 应用

1. 泛型函数返回值类型

   ```cpp
   template<typename Container>
   auto getValue(Container& c, int index) -> decltype(c[index]) {
       return c[index];  // 返回引用类型，可修改元素
   }
   ```



