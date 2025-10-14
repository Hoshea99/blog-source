---
title: static简述
date: 2025-10-10 16:45:12
categories: cpp
---

# static在全局作用域的意义

- 静态全局变量/函数

    ```c++
    //a.cpp
    static int counter = 0;
    static void helper();
    
    //b.cpp
    extern int counter; //无法访问static变量
    ```

静态全局变量可以限制符号的可见性为当前编译单元（.cpp）

# static在类内部的意义

- 静态成员变量

    ```c++
    class Counter{
        public：
            static int count;
    };

    int Counter::count = 0; //必须在cpp定义一次

    Counter c1,c2;
    c1.count = 5; //c2.count也等于5
    ```

  属于类本身，而不是类的实例，可以通过类名直接访问，也可以通过类的对象访问

- 静态成员函数

    ```c++
    class Logger{
        static void log(const std::string& msg){
            std::cout << msg << std::endl;
        }
    };
    ```

    没有this指针，不能访问非静态成员；可以访问其他静态成员

    不能作为虚函数
    
    可以通过类名直接调用

# static在函数内部的意义

- 静态局部变量
  
    ```c++
    void call_once(){
        static int called = 0; //只初始化这一次
        called++;
        cout << called;
    }
    call_once(); //输出1
    call_once(); //输出2
    ```

# 成员函数可以访问静态变量吗？

可以，但是静态成员函数不能访问普通变量。
