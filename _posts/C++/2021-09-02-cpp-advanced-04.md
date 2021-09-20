---
title: C++面向对象高级编程（4）参数传递与返回值
date: 2021-09-20 9:55:50
tags: C++
layout: post
---

### 构造函数放在private区的用法

这其实是单例模式实现的方法：

```C++
class A{
public:
    static A& getInstance();
    setup(){}
private:
    A();
    A(const A& rhs);
};

A& A::getInstance(){
    static A a;
    return a;
}
```





### 常量成员函数

在类的成员函数后面可以加 const 关键字，则该成员函数成为常量成员函数。

**对象加了const就是不要改类成员变量，const成员函数提供了这种保证。** 

**需要注意这跟平时用的其他const场景不太一样，这里在成员函数后加const是对所有成员的修改进行限制。**

这里有两个需要注意的点：

- 在常量成员函数中不能修改成员变量的值（静态成员变量除外）;
- 也不能调用同类的非 常量成员函数（静态成员函数除外）。

```C++
class Sample
{
public:
    void GetValue() const {} // 常量成员函数
    void func(){}
    int m_value;
};

void Sample::GetValue() const // 常量成员函数
{
    value = 0; // 出错，不能对成员变量和成员函数进行修改。
    func(); // 出错
}

int main()
{
    const Sample obj;
    obj.value = 100; // 出错，常量对象不可以被修改
    obj.func();    // 出错,常量对象上面不能执行 非 常量成员函数
    obj.GetValue;   // OK,常量对象上可以执行常量成员函数
    
    return 0;
}

```

**常引用**

引用前面可以加 const 关键字，成为常引用。不能通过常引用，修改其引用的变量的。

```javascript
const int & r = n;
r = 5; // error
n = 4; // ok!
```





### 参数传递 pass by value VS pass by reference(to const)

![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/arg_transfer.PNG)

返回值传递 pass by value VS pass by reference(to const)





## 友元函数

在学习c++这一块，关于友元函数和友元类，感觉还是不好理解，但是井下心来，理解，需要把我一下几点。
首先讲友元函数。

> 友元函数：

```
1)C++中引入友元函数，是为在该类中提供一个对外（除了他自己以外）访问的窗口;
2)这个友元函数他不属于该类的成员函数，他是定义在类外的普通函数，只是在类中声明该函数可以直接访问
```

使用友元函数注意的要点：

- 类中通过使用关键字friend 来修饰友元函数，但该函数并不是类的成员函数，其声明可以放在类的私有部分，也可放在共有部分。友元函数的定义在类体外实现，不需要加类限定。
- 一个类中的成员函数可以是另外一个类的友元函数，而且一个函数可以是多个类友元函数
- **友元函数可以访问类中的私有成员和其他数据，但是访问不可直接使用数据成员，需要通过对对象进行引用**
- **友元函数在调用上同一般函数一样，不必通过对对象进行引用。**



### 实例

```C++
//Simple program showing the working of the friend function:
#include <iostream>
using namespace std;

class Distance {
    private:
        int meter;
        // friend function
        friend int addFive(Distance);
    public:
        Distance() : meter(0) {}

};

// friend function definition
int addFive(Distance d) {
    //accessing private members from the friend function
    d.meter += 5;
    return d.meter;
}

int main() {
    Distance D;
    cout << "Distance: " << addFive(D);
    return 0;
}
```





```C++
//Adding members of two different classes using the friend function:

#include <iostream>

using namespace std;

// forward declaration

class ClassB;

class ClassA {

    public:
        ClassA() : numA(12) {}

    private:
        int numA;
      // friend function declaration
      friend int add(ClassA, ClassB);
};

class ClassB {
    public:
        ClassB() : numB(1) {}

    private:
        int numB;
        // friend function declaration
        friend int add(ClassA, ClassB);
};

// access members of both classes
int add(ClassA objectA, ClassB objectB) {
    return (objectA.numA + objectB.numB);
}

int main() {
    ClassA objectA;
    ClassB objectB;
    cout << "Sum: " << add(objectA, objectB);
    return 0;
}

```





```C++
//Printing the length of a box using the friend function:
#include <iostream>
using namespace std;

class Box
{
  private:
    int length;

  public:
    box(): length ( 0 ) { }
    friend int printLength(Box); // friend function
};

int printLength (Box b)
{
  b.length += 10;
   return b.length;
}

int main()
{
  Box b;
  Cout << “Length of box:” <<printLength(b)<<endl;
  Return 0 ;
}
```

### C++ 相同class的object互为友元

```C++
#ifndef __COMPLEX_H__
#define  __COMPLEX_H__
 
#include <iostream>
using namespace std;
 
class complex{
public:
	complex(double a=0, double b=0):re(a),rs(b)
	{}
    
    //这里直接访问传入参数的private数据
	double func(const complex& param)
	{
		return param.re + param.rs;
	}
 
private:
	double re, rs;
};
 
void Test_complex(){
	complex a1(2,1);
	complex a2;
	std::cout<<a2.func(a1);   //输出为3
	//std::cout<<a1.re;	//错误，不能访问
}
#endif
```

