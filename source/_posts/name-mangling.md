---
title: Name Mangling
date: 2025-10-11 15:44:09
categories: cpp
---

# 作用

解决函数重载的符号冲突，保持类型安全，区分作用域，模板实例化的区分

ABI规范定义了name mangling规则，确保了同一ABI的编译器能够互相链接。

由于C++支持重载，所以编译器要对函数名进行命名，但是MSVC和MINGW编译器的命名方法不同，会导致跨编译器编译出错，所以需要在代码上进行兼容。

# 方法

```cpp
// ImageSourceModuleAPI.h
#ifndef IMAGESOURCEMODULE_API_H
#define IMAGESOURCEMODULE_API_H
// DLL导出宏定义
#ifdef IMAGESOURCEMODULE_DLL_EXPORTS
    #define IMAGESOURCEMODULE_DLL_API __declspec(dllexport)
#else
    #define IMAGESOURCEMODULE_DLL_API __declspec(dllimport)
#endif
// 前向声明
class CAbstractUserModule;
#ifdef __cplusplus
extern "C"
{
#endif
 // 采用__stdcall调用约定，且须在.def文件中增加接口描述。
 IMAGESOURCEMODULE_DLL_API CAbstractUserModule* __stdcall CreateModule(void* hModule);
 IMAGESOURCEMODULE_DLL_API void __stdcall DestroyModule(void* hModule, CAbstractUserModule* pUserModule);
#ifdef __cplusplus
};
#endif
#endif // IMAGESOURCEMODULE_API_H
```

这种属于是extern "C"的方法，还有一种方法是在def文件里声明：

```def
; ImageSourceModule.def
EXPORTS
CreateModule
DestroyModule
```
