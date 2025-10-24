---
title: perf使用初探
date: 2025-10-24 15:21:15
categories: cpp
---

# perf简述

Perf是Linux内核自带的性能分析工具，基于硬件性能计数器，开销很小。



<!--more-->

## 安装

安装这里不再赘述，可以通过`perf --version`观察是否安装成功。

##  简单指令

### 快速性能统计

- `perf stat ./program`
- `perf stat -p <PID>`
- perf stat -a sleep 5

### 记录和分析性能数据

- 记录性能数据

  `perf record -g ./program`

- 查看分析结果

  `perf report`

- 图形化查看

  `perf report --tui`

# 实例分析

## 代码

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <chrono>
#include <thread>

// 1. 计算密集型函数 - 模拟热点
double heavy_computation(int iterations) {
    double result = 0.0;
    for (int i = 0; i < iterations; ++i) {
        result += std::sin(i * 0.1) * std::cos(i * 0.1);
    }
    return result;
}

// 2. 内存访问密集型函数
void memory_intensive() {
    std::vector<int> data(1000000);
    for (size_t i = 0; i < data.size(); ++i) {
        data[i] = i * 2;
    }
    
    // 模拟一些计算
    long sum = 0;
    for (size_t i = 0; i < data.size(); i += 100) {
        sum += data[i];
    }
}

// 3. 函数调用密集型
void function_a() { volatile int x = 1; }
void function_b() { volatile int y = 2; }
void function_c() { volatile int z = 3; }

void call_intensive() {
    for (int i = 0; i < 10000; ++i) {
        function_a();
        function_b();
        function_c();
    }
}

int main() {
    std::cout << "Perf 测试程序开始...\n";
    
    // 测试不同的工作负载
    auto start = std::chrono::high_resolution_clock::now();
    
    // 1. 计算密集型
    std::cout << "计算密集型任务...\n";
    double result1 = heavy_computation(5000000);
    
    // 2. 内存密集型  
    std::cout << "内存密集型任务...\n";
    memory_intensive();
    
    // 3. 函数调用密集型
    std::cout << "函数调用密集型任务...\n";
    call_intensive();
    
    // 4. 混合负载
    std::cout << "混合负载任务...\n";
    for (int i = 0; i < 1000; ++i) {
        heavy_computation(1000);
        memory_intensive();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "总执行时间: " << duration.count() << " ms\n";
    std::cout << "计算结果: " << result1 << "\n";
    std::cout << "测试完成!\n";
    
    return 0;
}
```

## 编译

`g++ -g perf.cpp -o test_perf`

注意要编译待调试信息的程序

## 调试

注意perf和待测试程序都需要加权限，我们这里测试为主，直接chmod 777即可

### 基础性能统计

`perf stat ./test_perf`

```plain
Perf 测试程序开始...
计算密集型任务...
内存密集型任务...
函数调用密集型任务...
混合负载任务...
总执行时间: 12109 ms
计算结果: 0.245091
测试完成!

 Performance counter stats for './test_perf':

          12071.67 msec task-clock                #    0.997 CPUs utilized
              5854      context-switches          #    0.485 K/sec
               117      cpu-migrations            #    0.010 K/sec
              2202      page-faults               #    0.182 K/sec
       27487318701      cycles                    #    2.277 GHz
       50921469069      instructions              #    1.85  insn per cycle
   <not supported>      branches
           1413851      branch-misses

      12.112368048 seconds time elapsed

      12.049445000 seconds user
       0.024992000 seconds sys

```

#### 基础指标

- 任务时钟：

  12071.67ms程序实际消耗的CPU时间，约为0.997核

- 耗时

  12.11秒，与任务时钟接近，说明CPU利用率很高

#### 关键性能指标

- 上下文切换

  5854次，每秒钟485次，属于中等水平

- CPU迁移

  117次，迁移次数较低，对性能影响较小

- 缺页异常

  2202次，主要为次要缺页，可能涉及动态内存分配

#### CPU核心指标

- 周期数

  274.87亿次

- 指令数，509.21亿条

- 每周期指令数IPC 1.85条 说明CPU执行效率良好

  解释：IPC越高，表示CPU在每个时钟周期内完成的有效工作越多，效率越高。

### 函数级分析

`./perf record -e cycles,instructions,cache-misses,cache-references -g ./test_perf`

- `./perf record`

  调用perf记录功能

- ` -e cycles,instructions,cache-misses,cache-references`

  指定检测性能事件：

  - cycles：cpu周期数
  - instructions：执行指令数
  - cache-missed和cache-references：缓存未命中次数和缓存访问总次数，结合可以计算出缓存未命中率

- -g

  记录每个样本当时的函数调用堆栈，可以看到哪个函数函数耗时多，还能知道谁调用了它
