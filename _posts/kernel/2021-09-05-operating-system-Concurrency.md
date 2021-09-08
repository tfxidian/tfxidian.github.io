---
title: Operating Systems Three Easy Pieces：并发
date: 2021-09-05 16:33:50
tags: linux, kernel, Concurrency
layout: post
---



### Concurrency: An Introduction



#### 为什么使用线程

事实证明，应该使用线程至少有两个主要原因。

第一个很简单:并行性。假设您正在编写一个程序，该程序在非常大的数组上执行操作，例如，将两个大数组相加，或将数组中每个元素的值增加一定数量。如果只在单个处理器上运行，那么任务很简单:只需执行每个操作并完成即可，但是可能速度比较慢。但是，如果您在一个有多个处理器的系统上执行程序，那么通过使用这些处理器，您就有可能大大提高这个进程的速度

第二个原因更微妙:避免由于缓慢的I/O而阻塞程序进程。假设您正在编写一个执行不同类型I/O的程序:等待发送或接收消息，等待磁盘I/O完成，甚至等待页面错误完成。与其等待，您的程序可能希望做一些其他的事情，包括利用CPU执行计算，甚至进一步发出I/O请求。使用线程是避免卡住的自然方法。

当然，**在上面提到的任何一种情况下，都可以使用多个进程而不是线程**。然而，线程共享一个地址空间，从而使共享数据变得容易，因此在构造这些类型的程序时是一个自然的选择。**对于逻辑上独立的任务，如果不需要共享内存中的数据结构，则进程是更合理的选择。**

#### 一个简单的线程实例

```C
#include <stdio.h>
#include <assert.h>
#include <pthread.h>

void* mythread(void *arg){
    printf("%s\n", (char *)arg);
    return NULL;
}

int main(){
    pthread_t p1, p2;
    int rc;
    printf("this is first thread test\n");
    pthread_create(&p1, NULL, mythread, "A");
    pthread_create(&p2, NULL, mythread, "B");
    pthread_join(p1,NULL);
    pthread_join(p2,NULL);
    printf("the end!\n");
    return 0;

}
```

执行命令`gcc -pthread test.c -o test` 运行结果如下：

```c
this is first thread test
A
B
the end!
```

下面是单一线程和多线程的内存分布：

![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/thread1.PNG)

在一个多线程进程中，每个线程都独立运行。在该图中，可以看到两个栈分布在进程的地址空间中。因此，任何栈分配的变量、参数、返回值和我们放在栈上的其他东西都将被放线程相关的栈上。



创建线程看上去有点像调用函数，但是，系统**不是首先执行函数然后返回到调用者，而是为被调用的函数创建一个新的执行线程，并且它独立于调用者运行**，接下来怎么运行就取决于操作系统的调度。



#### Why It Gets Worse: Shared Data

我们上面展示的简单线程示例有助于说明如何创建线程，以及如何根据调度程序决定如何运行线程。但是，它没有展示线程在访问共享数据时是如何交互的。

下面我们看一个例子：

```C
#include<stdio.h>
#include<pthread.h>

static volatile int counter = 0;

void* mythread(void* arg){
    printf("%s\n", (char*)arg);
    int i;
    for(i = 0; i< 1000000; i++){
        counter = counter + 1;
    }
    return NULL;
}

int main(){
    pthread_t t1, t2;
    pthread_create(&t1, NULL, mythread, "A");
    pthread_create(&t2, NULL, mythread, "B");
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("main: done with both counter = %d\n", counter);
    return 0;
}
```

在这个例子中，线程执行部分时间要长一些，可以比较明显的从输出结果中看到问题。

执行结果输出：

```
tf@LAPTOP-ORMEC5DB:~$ ./sharedata 
A
B
main: done with both counter = 1161833
tf@LAPTOP-ORMEC5DB:~$ ./sharedata 
A
B
main: done with both counter = 1064672
tf@LAPTOP-ORMEC5DB:~$ ./sharedata 
A
B
main: done with both counter = 1007078
tf@LAPTOP-ORMEC5DB:~$ ./sharedata 
A
B
main: done with both counter = 1041353
```

可以说基本上每次都不一样，那么为什么会出现这样的结果呢？



#### 问题的核心:不受控制的线程调度

要理解为什么会发生这种情况，我们必须理解编译器为计数器更新生成的代码序列。在上面的例中，我们希望简单地向counter添加一个数字1, 执行此操作的代码序列可能如下所示(在x86中):

```assembly
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

一个简单的加1操作实际上分为3步：

1. 从内存0x8049a1c中取值放到eax寄存器，
2. 寄存器值加1
3. 把寄存器加1后的值放回内存0x8049a1c

那么就可能出现这样的情况：两个线程（几乎）同时执行了第1步，然后第1个进程和第二个进程的加1操作都是在第1步取值基础上进行的，那么虽然两个进程都加了1，可效果跟加1次是一样的。比如当前内存中counter的值为50，进程1和进城2都取了值为50，然后后面的操作二者也都加1，结果得到了51，跟预想中的52完全不同。



我们在这里演示的被称为竞争条件(或者更具体地说，数据竞争)，当各个线程访问数据资源时会出现竞争状态，即：数据几乎同步会被多个线程占用，造成数据混乱。事实上，我们每次可能得到不同的结果。

因为执行此代码的多个线程可能会导致竞争条件，所以我们称此代码为临界区。**临界段是访问共享变量(或者更普遍地说，访问共享资源)的一段代码，并且不能由多个线程并发执行。**

我们真正想要的是所谓的**互斥**：这个属性保证，如果一个线程在临界区内执行，其他线程将无法执行。

#### 原子性的愿望

解决这个问题的一种方法是使用更强大的指令，在一个步骤中，完成我们需要完成的任何事情，从而消除了不及时中断的可能性。例如，如果我们有一个像这样的超级指令:

```
memory-add 0x8049a1c, $0x1
```

这是美好的愿望，现实中我们通过使用硬件支持，并结合操作系统的一些帮助，构建多线程代码，以同步和控制的方式访问关键部分，从而可靠地产生正确的结果，尽管并发执行具有挑战性。

### Thread API

### Locks

#### Locks: The Basic Idea



### Locked Data Structures

### Condition Variables 

### Semaphores

### Concurrency Bugs

### Event-based Concurrency

### Summary