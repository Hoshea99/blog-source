---
title: cmake
date: 2025-10-13 14:43:52
categories: cpp
---

# 生成静态/动态链接库

## 目录

```plain
    math_library/
    ├── CMakeLists.txt
    ├── include/
    │   └── math_utils.h
    └── src/
        └── math_utils.cpp
```

## 代码

**头文件：**

```cpp
#pragma once

int add(int a,int b);
int multiply(int a,int b);
```

**源文件：**

```cpp
#include "math_utiles.h"

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}
```

**Cmake:**

```Cmake
cmake_minimum_required(VERSION 3.10)
project(MathLibrary VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)

include_directories(include)

add_library(math_shared SHARED src/math_utils.cpp)
add_library(math_static STATIC src/math_utils.cpp)
```

# 写一个程序用到上面的动态库

**目录：**

```plain
project_root/
├── math_library/
│   ├── include/
│   │   └── math_utiles.h
│   ├── src/
│   │   ├── math_utiles.cpp
│   │   └── CMakeLists.txt
│   ├── lib/
│   │   └── libmath_shared.so
│   └── CMakeLists.txt
├── math_test/
│   ├── CMakeLists.txt
│   └── test_math.cpp
└── CMakeLists.txt (顶层)
```

**源文件：**

```cpp
#include<iostream>
#include"math_utiles.h"

int main(){
    std::cout<<"3+4="<<add(3,4)<<std::endl;
    std::cout<<"3*4="<<multiply(3,4)<<std::endl;
    return 0;
}
```

**Cmake:**

```Cmake
cmake_minimum_required(VERSION 3.10)
project(MathTest)

# 设置库文件路径
set(MATH_UTILS_LIB "${CMAKE_CURRENT_SOURCE_DIR}/libmath_shared.so")

# 创建测试程序
add_executable(test_math test_math.cpp)

# 链接库并包含头文件
target_link_libraries(test_math PRIVATE ${MATH_UTILS_LIB})
target_include_directories(test_math PRIVATE 
    ${CMAKE_CURRENT_SOURCE_DIR}/../math_library/include
)

# 如果库不在当前目录，需要复制或创建符号链接
if(NOT EXISTS ${MATH_UTILS_LIB})
    message(WARNING "libmath_shared.so not found in current directory")
endif()
```

# Cmake问题

## PRIVATE PUBLIC INTERFACE区别

- public

  A和依赖A的目标都能使用

- private

  仅A使用

- interface

  仅依赖A的目标能使用

## 动态链接库对比静态链接库有哪些好处

1. 节省磁盘空间
2. 更新维护时不需要重新编译
3. dlopen可以按需加载
