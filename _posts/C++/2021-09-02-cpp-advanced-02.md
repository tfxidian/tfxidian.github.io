---
title: C++面向对象高级编程（2）头文件与类的声明
date: 2021-09-20 9:55:50
tags: C++
layout: post
---



## 基于对象与面向对象的区别

基于对象（Object Based）：面对的是单一class的設計

面向对象（Object Oriented）：面对的是多重classes 的设计，classes 和classes 之间的关系。

显然，要写好面向对象的程序，先基于对象写出单个class是比不可少的。

## C++类的两个经典分类

一个是没有指针的类，比如将要写的complex类，只有实部和虚部，另一个就是带有指针的类，比如将要写的另一个类string，数据内部只有一个指针，采用动态分配内存，该指针就指向动态分配的内存。



## 头文件防卫式声明

![](https://github.com/FangYang970206/Cpp-Notes/blob/master/C%2B%2B%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E9%AB%98%E7%BA%A7%E7%BC%96%E7%A8%8B/assets/1557456561799.png)

防卫式声明，与c语言一样，防止头文件重复包含，上面是经典写法，还有一个`# pragma once`的写法，两者的区别可以参考这篇[博客](https://blog.csdn.net/winscar/article/details/7016146)。



## 简单框架

```C++
//main.cpp
#include <iostream>
#include "Complex.h"
using namespace std;

int main()
{
    cout << "Hello world!" << endl;

    complex<double> c1(2.5,1.5);
    complex<int> c2(2,6);
    return 0;
}

```



```C++
//Complex.h
#ifndef __COMPLEX__
#define __COMPLEX__
#include <cmath>
//class complex;

template<typename T>
class complex{

public:
    complex(T r = 0, T i = 0)
        :re(r),im(i)
    { }
    complex& operator += (const complex&);
    T real() const{return re;}
    T imag() const {return im;}

private:
    T re, im;
    friend complex& __doap(complex*, const complex&);
};

#endif // __COMPLEX__
```

