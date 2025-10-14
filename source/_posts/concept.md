---
title: concept
date: 2025-10-10 16:37:05
categories: cpp
---

Concept是C++20引用的一种对模板参数进行约束的机制，它允许程序员明确指定模板参数必须满足的要求，从而在编译期就能捕获不符合要求的模板参数错误。

# 语法基础

- 定义
  
  ```cpp
    template<typename T>
    concept MyConcept = requires(T a,T b){
        {a+b}->std::convertible_to<T>;
        a.size();
    };
  ```

- 使用

    ```cpp
    template<MyConcept T>
    void myFunction(T param){
    
    }
    ```

标准库中存在很多预定义的concept，在`<concepts>`头文件中。

# requires

requires表达式用于定义Concept的具体要求。子句类型有如下几种：

- 简单要求  
    只要求表达式是否合法：`{a+b}`

- 类型要求  
    检查类型是否存在:`typename T::value_type;`

- 符合要求  
    检查表达式属性：`{a+b}->std::same_as<T>;`

- 嵌套要求  
    在requires表达式内使用constexpr布尔表达式：`requires sizeof(T)>4;`

# 应用场景

应用场景包括约束函数模板、类模板、约束auto变量、约束返回类型、约束多个参数。
