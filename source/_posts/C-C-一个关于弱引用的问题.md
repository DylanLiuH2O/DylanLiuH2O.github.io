---
title: C/C++一个关于弱引用的问题
date: 2021-04-11 22:57:59
tags: [C/C++, 链接]
categories: 问题记录
---

最近在看《程序员的自我修养——链接、装载与库》，在读了关于强符号、弱符号、强引用以及弱引用相关的部分后，按照书上的示例写了段代码，但结果没有达到预期。

### 强符号与弱符号
对于C/C++来说，编译器默认函数与初始化了的全局变量为**强符号**，未初始化的全局变量为**弱符号**。

### 强引用与弱引用

目标文件在被链接成为可执行文件的时，那些文件引用的外部符号都要被正确决议，也就是在别的文件中找到它的定义，如果没有找到就会报未定义的错误，这称为**强引用**。

相对的，还有一种叫**弱引用**，处理弱引用时，如果能找到符号的定义，那么就引用它，如果找不到，则不处理也不报错，执行的安全性由用户来保证（也就是先判断是否有值）。

在GCC中，可以使用`__attribute__((weak))`来显式声明对一个外部函数的引用为弱引用。

### 例程

书上给出了如下的程序

```c
#include <stdio.h>
#include <pthread.h>

int pthread_create(
    pthread_t*, 
    const pthread_attr_t*, 
    void* (*)(void*), 
    void*)__attribute__((weak));

int main(void)
{
    if(pthread_create) {
        printf("This is multi-thread version!\n"); 
    } else {
        printf("This is single-thread version!\n"); 
    }
}
```

预期结果应该是

```
$ gcc phtread.c -o pt
$ ./pt
This is single-thread version!
$ gcc phtread.c -lpthread -o pt
$ ./pt
This is multi-thread version!
```

但实测我的结果是：

![res1](res1.png)

此时内心黑人问号，于是我打印了一下函数的地址：
```c
#include <stdio.h>
#include <pthread.h>

int pthread_create(
    pthread_t*, 
    const pthread_attr_t*, 
    void* (*)(void*), 
    void*)__attribute__((weak));

int main(void)
{
    printf("pthread_create: %p\n", pthread_create);
    if(pthread_create) {
        printf("This is multi-thread version!\n"); 
    } else {
        printf("This is single-thread version!\n"); 
    }
}
```

![res2](res2.png)

看来并没有正确链接到，接着我查看了一下加了`-lpthread`链接选项的可执行文件的符号表：

![symbol_list](symbol_list.png)

果不其然，符号`pthread_create`是UNDEFINED（未定义）的。

接着我把弱引用的修饰去掉，看看能不能正常链接：

```c++
#include <stdio.h>
#include <pthread.h>

int pthread_create(
    pthread_t*, 
    const pthread_attr_t*, 
    void* (*)(void*), 
    void*);

int main(void)
{
    printf("pthread_create: %p\n", pthread_create);
    if(pthread_create) {
        printf("This is multi-thread version!\n"); 
    } else {
        printf("This is single-thread version!\n"); 
    }
}
```

![res3](res3.png)

结果是能正确链接了，但这让我更迷惑了，到底为什么声明弱引用之后就无法正确链接了呢？

这方面的资料着实有点少，找了一些还是没有找到答案，那么就先放在这里吧，以后说不定会找到答案。

如果你知道问题所在的话，烦请告诉我，在我的主页有我的联系方式，十分感谢！！