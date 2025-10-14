---
title: packaged_task
date: 2025-10-13 22:43:08
categories: cpp
---

`packaged_task` 是 C++11 引入的一个重要组件，它可以将可调用对象（函数、lambda、函数对象等）包装起来，并将其返回值与一个 `future` 对象关联，从而方便地获取异步操作的结果。

# 创建

```cpp
// 包装普通函数
int add(int a, int b) {
    return a + b;
}

packaged_task<int(int, int)> task(add);
```

# 执行任务

```cpp
// 在另一个线程中执行
thread t(move(task), 10, 20);
t.join();

// 获取结果
cout << "Result: " << result.get() << endl; // 输出: Result: 30
```

# 特性

## 移动语义

智能移动不能复制

## 重置任务

```cpp
packaged_task<int()> task([]{ return 42; });
task(); // 第一次执行

// 任务已执行，需要重置才能再次使用
task.reset(); // 重置任务状态
future<int> fut2 = task.get_future();
task(); // 可以再次执行
```

# 类型擦除

可以看线程池一文的第二种实现方法
