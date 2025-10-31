---
title: 手写memcpy（逐步优化）
date: 2025-10-30 15:05:29
categories: cpp
---

手写一个memcpy函数考察了对指针操作、内存对齐和性能优化的理解。

# 基础版本（逐字节拷贝）

```cpp
void* my_memcpy_basic(void* dest,const void* src,size_t n){
    if(dest == NULL || src ==NULL){
        return NULL;
    }
    char* d = (char*)dest;
    const char* s = (const char*) src;
    
    for(size_t i = 0;i<n;i++){
        d[i]=s[i];
    }
    
    return dest;
}
```

这个版本只拷贝一个字节，循环次数多，效率比较低。而且没有考虑到内存重叠的问题。

# 改进版本 （考虑到内存重叠）

如果源地址在目标地址之前，且存在重叠情况，从前向后拷贝会破坏源数据。

```cpp
void* my_memcpy(void* dest, const void* src, size_t n) {
    if (dest == NULL || src == NULL) {
        return NULL;
    }

    char* d = (char*)dest;
    const char* s = (const char*)src;

    // 检查内存是否重叠
    if (s < d && s + n > d) {
        // 存在重叠，且源地址在目标地址之前。需要从后向前拷贝以避免覆盖。
        d += n - 1;
        s += n - 1;
        while (n--) {
            *d-- = *s--;
        }
    } else {
        // 无重叠，或源地址在目标地址之后。从前向后拷贝。
        while (n--) {
            *d++ = *s++;
        }
    }

    return dest;
}
```

# 优化版本 （按机器字长拷贝）

为了追求性能，可以尝试按CPU一次可以处理的最大位数进行拷贝，然后处理零头。

```cpp
void* my_memcpy_fast(void* dest, const void* src, size_t n) {
    if (dest == NULL || src == NULL) {
        return NULL;
    }

    // 使用 uintptr_t 作为机器字长的类型
    uintptr_t* d_word = (uintptr_t*)dest;
    const uintptr_t* s_word = (const uintptr_t*)src;

    // 计算能按字长拷贝多少次
    size_t word_count = n / sizeof(uintptr_t);
    // 计算剩下的字节数
    size_t byte_remain = n % sizeof(uintptr_t);

    // 按字长拷贝
    for (size_t i = 0; i < word_count; i++) {
        d_word[i] = s_word[i];
    }

    // 处理剩余的字节
    if (byte_remain > 0) {
        // 定位到已拷贝字长之后的地址
        // 注意：这里转换回 uint8_t* (或 char*) 进行逐字节操作
        uint8_t* d_byte = (uint8_t*)d_word + word_count * sizeof(uintptr_t);
        const uint8_t* s_byte = (const uint8_t*)s_word + word_count * sizeof(uintptr_t);
        for (size_t i = 0; i < byte_remain; i++) {
            d_byte[i] = s_byte[i];
        }
    }

    return dest;
}
```

然而这个版本有个问题：即假定了dest和src进行了正确对齐。如果地址没对齐，在arm架构会导致程序崩溃，在x86会导致性能下降。

# 优化版本（处理数据对齐）

```cpp
void* my_memcpy_optimized(void* dest, const void* src, size_t n) {
    if (dest == NULL || src == NULL || n == 0) {
        return dest;
    }

    uint8_t* d = (uint8_t*)dest;
    const uint8_t* s = (const uint8_t*)src;

    // 1. 对于小数据量的优化：如果数据量很小，直接使用逐字节拷贝更划算
    // 避免了块操作带来的循环和判断开销。阈值可以调整（例如8或16字节）。
    const size_t SMALL_SIZE_THRESHOLD = 16;
    if (n < SMALL_SIZE_THRESHOLD) {
        // 使用简单的循环进行逐字节拷贝
        for (size_t i = 0; i < n; i++) {
            d[i] = s[i];
        }
        return dest;
    }

    // 2. 对齐目标地址 (d)
    // 计算当前地址 d 距离下一个 uintptr_t 对齐地址的偏移量
    // 例如：如果 d 是 0x1001, uintptr_t 是 4 字节，则：
    // (uintptr_t)d % 4 = 1
    // 4 - 1 = 3，所以需要拷贝 3 个字节来对齐。
    size_t align_offset = (sizeof(uintptr_t) - ((uintptr_t)d % sizeof(uintptr_t))) % sizeof(uintptr_t);

    // 先按字节拷贝这前 align_offset 个字节，使 d 对齐
    for (size_t i = 0; i < align_offset; i++) {
        d[i] = s[i];
    }

    // 更新指针和剩余字节数
    d += align_offset;
    s += align_offset;
    n -= align_offset;

    // 3. 现在目标地址 d 已经对齐。按机器字长 (uintptr_t) 进行块拷贝。
    uintptr_t* d_word = (uintptr_t*)d;
    const uintptr_t* s_word = (const uintptr_t*)s;
    size_t word_count = n / sizeof(uintptr_t);

    for (size_t i = 0; i < word_count; i++) {
        d_word[i] = s_word[i];
    }

    // 4. 计算块拷贝后剩余的位置和字节数
    size_t bytes_copied_by_words = word_count * sizeof(uintptr_t);
    d = (uint8_t*)d_word + bytes_copied_by_words;
    s = (const uint8_t*)s_word + bytes_copied_by_words;
    n -= bytes_copied_by_words;

    // 5. 处理最后的剩余字节
    for (size_t i = 0; i < n; i++) {
        d[i] = s[i];
    }

    return dest;
}
```

需要注意的是，按机器字长拷贝的话，就比较难考虑内存重叠问题了。因此如果使用方案4，要确保内存是没有重叠问题的。
