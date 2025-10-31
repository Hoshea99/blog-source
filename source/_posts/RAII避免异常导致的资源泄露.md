---
title: RAII避免异常导致的资源泄露
date: 2025-10-29 15:08:50
categories: cpp
---

在C++中，使用RAII来管理动态内存，其本身就是为了从根本避免异常导致的资源泄露。关键在于：将资源的生命周期与对象的生命周期绑定，当对象离开其作用域时（无论是正常还是异常栈展开），他的析构函数都会被自动调用，从而确保资源被释放。

<!--more-->

# 文件句柄RAII

使用智能指针的方法讲了数次了，这里不展开了。RAII的思想不限于内存，还包括文件句柄、套接字等。这里举例一个文件句柄：

```cpp
class FileRAII {
private:
    FILE* file_ptr;
public:
    // 构造函数获取资源
    explicit FileRAII(const char* filename, const char* mode) 
        : file_ptr(std::fopen(filename, mode)) {
        if (!file_ptr) {
            throw std::runtime_error("Failed to open file");
        }
    }

    // 析构函数释放资源
    ~FileRAII() {
        if (file_ptr) {
            std::fclose(file_ptr);
        }
    }

    // 禁止拷贝（或实现深拷贝/移动语义）
    FileRAII(const FileRAII&) = delete;
    FileRAII& operator=(const FileRAII&) = delete;

    // 可选：提供访问原始资源的接口
    FILE* get() const { return file_ptr; }
};

void useFile() {
    FileRAII file("data.txt", "r"); // 文件被打开
    someFunctionThatMightThrow();   // 可能抛出异常
    // 读取文件...
} // 无论是否异常，file的析构函数都会关闭文件 -> 无资源泄漏！

```

# 崩溃

RAII处理内存释放问题是基于程序正常的控制流的，当系统出现段异常等非正常终止情况时，RAII也没有办法。现代操作系统（如Linux, Windows）在进程终止时，会回收该进程拥有的所有内存和大多数内核资源（如文件描述符、套接字）。所以，对于内存和这些内核资源，即使崩溃也不会造成“永久性泄漏”。

但是，如果是数据库事务等资源，则需要人们从系统架构层面考虑。

# 总结

- 核心原则：将资源获取置于构造函数内，将资源释放置于析构函数内。
- 依赖栈展开：信任C++的异常机制。当异常抛出时，系统会析构所有已成功构造的局部RAII对象，从而自动释放其管理的资源。
- 优先使用标准库工具：对于动态内存，首选 std::unique_ptr 和 std::shared_ptr，并使用 make_unique/make_shared 来构造它们。这是最简单、最安全的方式。
- 为其他资源自定义RAII类：对于非内存资源，遵循同样的模式封装成类，从而将琐碎且易错的资源管理任务转化为可靠的、自动化的对象生命周期管理。
