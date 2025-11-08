---
title: allocator
date: 2025-10-31 16:08:02
categories: cpp
---

# 概述

Allocator是C++标准库中用于封装内存管理细节的类模板，他负责内存的分配、释放、对象的构造和析构。

**核心思想：**

将内存分配策略和数据结构分离开。

# 作用与价值

- 灵活性： 

  允许用户为STL容器自定义内存来源。例如：

  - 性能优化：使用内存池、栈上内存、共享内存等，避免频繁的系统调用（如malloc）。
  - 特殊场景：在嵌入式系统中使用静态内存，或在多线程环境中使用线程局部的内存池以避免锁竞争。

- 控制权： 让开发者能够精确控制内存的布局和对齐方式。
- 类型透明： 标准要求`allocator<T>`的rebind机制，使得为一个类型T分配的分配器可以用于分配另一种类型U的内存（这在std::list等节点式容器中至关重要）。

# 什么时候需要自定义Allocator

1. 高性能需求：比如内存池和无锁
2. 资源受限：使用预分配的内存块，避免动态分配
3. 特殊内存区域：需要内存对齐或者共享内存
4. 调试与检测：在分配与释放时记录信息

# 注意点

1. 策略模式
2. rebind机制
3. allocator_trait

# 简单实现

```cpp
#include <cstdlib>   // for std::size_t, std::ptrdiff_t
#include <new>       // for std::bad_alloc, placement new
#include <iostream>  // for demo

// 一个最简单的符合C++11标准的分配器
template <class T>
class SimpleAllocator {
public:
    // 1. 必需的类型定义
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;

    // 告诉容器：在容器拷贝/移动/交换时，这个分配器也应该被拷贝/移动/交换
    using propagate_on_container_copy_assignment = std::true_type;
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap = std::true_type;

    // 2. 默认构造函数（可能被容器使用）
    SimpleAllocator() = default;

    // 3. 模板拷贝构造函数：允许从 SimpleAllocator<U> 构造 SimpleAllocator<T>
    template <class U>
    SimpleAllocator(const SimpleAllocator<U>&) noexcept {}

    // 4. 核心功能：分配内存（只分配，不构造对象）
    T* allocate(std::size_t n) {
        if (n > std::size_t(-1) / sizeof(T)) {
            throw std::bad_alloc(); // 请求的内存过大
        }
        
        // 使用 ::operator new 分配原始内存
        if (auto p = static_cast<T*>(::operator new(n * sizeof(T)))) {
            std::cout << "Allocated " << (n * sizeof(T)) << " bytes at " << p << std::endl;
            return p;
        }
        throw std::bad_alloc();
    }

    // 5. 核心功能：释放内存（假定内存中的对象已经被destroy）
    void deallocate(T* p, std::size_t n) noexcept {
        std::cout << "Deallocating " << (n * sizeof(T)) << " bytes at " << p << std::endl;
        ::operator delete(p);
    }

    // 6. 可选：构造对象（在已分配的内存上）
    template <class U, class... Args>
    void construct(U* p, Args&&... args) {
        std::cout << "Constructing object at " << p << std::endl;
        ::new((void*)p) U(std::forward<Args>(args)...); // placement new
    }

    // 7. 可选：析构对象（不释放内存）
    template <class U>
    void destroy(U* p) {
        std::cout << "Destroying object at " << p << std::endl;
        p->~U();
    }
        // 8. 最重要的机制：rebind
    // 允许容器获取用于分配其他类型（如链表节点）的分配器
    template <class U>
    struct rebind {
        using other = SimpleAllocator<U>;
    };
};

// 9. 比较操作符：这个简单的分配器是无状态的，所以总是相等
template <class T, class U>
bool operator==(const SimpleAllocator<T>&, const SimpleAllocator<U>&) {
    return true;
}

template <class T, class U>
bool operator!=(const SimpleAllocator<T>& lhs, const SimpleAllocator<U>& rhs) {
    return !(lhs == rhs);
}

// 演示如何使用
int main() {
    // 使用我们的自定义分配器创建一个vector
    std::vector<int, SimpleAllocator<int>> vec;
    
    std::cout << "=== Pushing back 3 elements ===" << std::endl;
    vec.push_back(1);
    vec.push_back(2);
    vec.push_back(3); // 这里可能会触发重新分配
    
    std::cout << "Vector contents: ";
    for (int x : vec) {
        std::cout << x << " ";
    }
    std::cout << std::endl;
    
    std::cout << "=== Vector goes out of scope ===" << std::endl;
    // vec析构时，会自动调用allocator的destroy和deallocate
    return 0;
}
```



目前用的还是new和delete，没有明显作用，如果要有实际价值，需要修改函数：

```cpp
// 示例：改为从内存池分配
T* allocate(std::size_t n) {
    return static_cast<T*>(my_memory_pool.allocate(n * sizeof(T)));
}

void deallocate(T* p, std::size_t n) {
    my_memory_pool.deallocate(p, n * sizeof(T));
}
```

需要注意的是，我们不需要继承任何基类。

## value_type

注意到代码中有这些部分：

```cpp
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
```

这些是为了在vector中使用这些声明：

```cpp
template <class T, class Allocator = std::allocator<T>>
class vector {
private:
    // 容器内部会使用你定义的类型！
    using pointer = typename Allocator::pointer;
    using const_pointer = typename Allocator::const_pointer;
    using size_type = typename Allocator::size_type;
    
    pointer m_data;          // 使用你的pointer类型
    size_type m_size;        // 使用你的size_type
    size_type m_capacity;    // 使用你的size_type
    
public:
    // 返回的迭代器类型也依赖于这些定义
    using iterator = pointer;
    using const_iterator = const_pointer;
    
    // 成员函数签名也使用这些类型
    pointer data() noexcept { return m_data; }
    const_pointer data() const noexcept { return m_data; }
    size_type size() const noexcept { return m_size; }
};
```

> 这是我们就有一个疑问，为什么不直接在vector设置，而是套用一层allocator呢？

这是因为，为了支持任意的非标准的分配器。比如我们的输出的指针类型可能是带有调试信息的指针：

```cpp
template<class T>
class DebugAllocator {
public:
    // 不是返回普通的T*，而是返回一个包装类型！
    using pointer = DebugPointer<T>;  // 自定义的智能指针类型
    using size_type = DebugSizeType;  // 带边界检查的size类型
    
    pointer allocate(size_type n) {
        // 返回的不是T*，而是DebugPointer<T>
        return DebugPointer<T>(...);
    }
};

// vector必须使用分配器定义的pointer类型！
std::vector<int, DebugAllocator<int>> vec;
auto p = vec.data();  // p的类型是DebugPointer<int>，不是int*
```

**这是我们又有个疑问，每个Allocator都需要用这一套声明吗？**

也不是，由于trait的存在，我们其实只要定义一个`value_type`即可：

```cpp
#include <memory>  // 包含 std::allocator_traits

// 你的最简化分配器
template <class T>
class MinimalAllocator {
public:
    using value_type = T;
    
    T* allocate(std::size_t n) {
        std::cout << "MinimalAllocator::allocate(" << n << ")" << std::endl;
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    
    void deallocate(T* p, std::size_t n) {
        std::cout << "MinimalAllocator::deallocate(" << p << ", " << n << ")" << std::endl;
        ::operator delete(p);
    }
};

// vector内部的简化实现（展示关键部分）
template<typename T, typename Allocator = MinimalAllocator<T>>
class MyVector {
private:
    T* data_ = nullptr;
    std::size_t size_ = 0;
    std::size_t capacity_ = 0;
    
    // 关键：使用allocator_traits！
    using alloc_traits = std::allocator_traits<Allocator>;
    Allocator allocator_;  // 分配器实例
public:
    MyVector() = default;
    
    void push_back(const T& value) {
        if (size_ >= capacity_) {
            // 需要扩容
            std::size_t new_capacity = capacity_ == 0 ? 1 : capacity_ * 2;
            
            // 通过allocator_traits分配内存
            T* new_data = alloc_traits::allocate(allocator_, new_capacity);
            std::cout << "Allocated new memory at: " << new_data << std::endl;
            
            // 通过allocator_traits构造对象（移动现有元素）
            for (std::size_t i = 0; i < size_; ++i) {
                alloc_traits::construct(allocator_, &new_data[i], std::move(data_[i]));
            }
            
            // 通过allocator_traits析构旧对象
            for (std::size_t i = 0; i < size_; ++i) {
                alloc_traits::destroy(allocator_, &data_[i]);
            }
            
            // 通过allocator_traits释放旧内存
            if (data_) {
                alloc_traits::deallocate(allocator_, data_, capacity_);
            }
            
            data_ = new_data;
            capacity_ = new_capacity;
        }
        
        // 构造新元素
        alloc_traits::construct(allocator_, &data_[size_], value);
        ++size_;
    }
    
    ~MyVector() {
        // 析构所有对象
        for (std::size_t i = 0; i < size_; ++i) {
            alloc_traits::destroy(allocator_, &data_[i]);
        }
        
        // 释放内存
        if (data_) {
            alloc_traits::deallocate(allocator_, data_, capacity_);
        }
    }
};

// 使用示例
int main() {
    MyVector<int, MinimalAllocator<int>> vec;
    vec.push_back(42);
    vec.push_back(100);
    return 0;
}
```

allocator_traits的关键作用就是提供方法的默认实现和类型推导。如果自定义的不存在，则用默认的，这块设计使用到了SFINAE

































