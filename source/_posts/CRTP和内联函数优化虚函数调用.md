---
title: CRTP和内联函数优化虚函数调用
date: 2025-10-29 16:14:53
categories: cpp
---

# 虚函数的开销

虚函数是C++实现运行时多态的基石，但它确实会带来性能开销：

1. 虚函数表（vtable）查找：每次调用虚函数，都需要通过对象的vptr（虚函数表指针）找到vtable，再通过索引找到正确的函数地址，然后进行调用。这比直接函数调用多了一次间接寻址。
2. 无法内联：由于具体调用哪个函数在运行时才能确定，编译器在编译期无法将函数体直接嵌入到调用处，因此无法进行内联优化。内联可以消除函数调用开销，并且为后续优化（如常量传播）打开大门。
3. 缓存不友好：间接跳转可能导致CPU指令缓存预测失败。

<!--more-->

# CRTP

已经介绍过，[链接地址](https://hoshea99.github.io/2025/10/11/crtp/)

# 内联函数

其实就是直接静态分派去替代虚函数

```cpp
// 为不同类型提供特化/重载的函数，而不是类
namespace processor {
    void process(const DataA& data) { /* 针对A的优化实现 */ }
    void process(const DataB& data) { /* 针对B的优化实现 */ }
}

// 使用 std::variant 或 if-constexpr 进行分派
using DataVariant = std::variant<DataA, DataB>;

// 方式1：使用std::visit (编译时分派)
void process_heterogeneous_list(const std::vector<DataVariant>& data_list) {
    for (const auto& item : data_list) {
        // 使用 std::visit 在运行时根据 item 实际存储的类型来分派
        std::visit([](const auto& d) {
            processor::process(d); // 这里会发生编译时分派！
        }, item);
    }
}

// 方式2：如果类型在编译期已知，直接调用（最强优化）
template <typename DataType>
void hot_loop(const std::vector<DataType>& data_list) {
    for (const auto& item : data_list) {
        processor::process(item); // 直接调用，100%可内联
    }
}
```

该方法有效地消除了函数调用开销。第一种方法是这个vector里可能有不同类型，第二种方法是这个vector里只有一种类型
