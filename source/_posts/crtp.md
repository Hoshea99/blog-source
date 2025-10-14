---
title: CRTP
date: 2025-10-11 15:10:56
categories: cpp
---

# CRTP语法

```cpp
template<typename Derived>
class Base {
public:
    void interface() {
        // 静态向下转换到派生类
        static_cast<Derived*>(this)->implementation();
    }
};

class Derived : public Base<Derived> {
public:
    void implementation() {
        // 具体实现
        std::cout << "Derived implementation" << std::endl;
    }
};
```

# CRTP的作用

无虚函数开销，适用于静态时多态。具有性能优势和编译时多态的特点。

# 风险

基类使用派生类成员如果没有这个方法，编译会错误。解决方法为使用Concept

```cpp
template<typename T>
concept Drawable = requires(T t){
    t.draw();
};

template<Drawable Derived>
class GraphicsBase{
    //...
};
```
