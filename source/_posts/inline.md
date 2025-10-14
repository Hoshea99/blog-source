---
title: inline
date: 2025-10-11 15:49:31
categories: cpp
---

# 原理

```cpp
// 声明为inline的函数
inline int add(int a, int b) {
    return a + b;
}

int main() {
    int result = add(3, 4);  // 可能被内联
    return 0;
}
```

内联后

```cpp
int main() {
    int result = 3 + 4;  // 直接替换函数调用
    return 0;
}
```

# 编译器不内联的情况

1. 函数体过于复杂
2. 递归
3. 包含静态变量
4. 虚函数
5. 函数指针调用
