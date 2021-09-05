---
title: malloc tutorial
date: 2021-09-05 16:33:50
tags: linux, kernel
layout: post
---



让我们编写一个malloc，看看它如何与现有程序一起工作!

首先，malloc的函数签名是void *malloc(size_t size);我们自己写一个名为malloc2的内存分配函数。

操作系统为进程保留堆栈和堆空间，sbrk让我们可以操纵堆。sbrk(0)返回一个指向当前堆顶的指针。sbrk (size)按size增加堆大小，并返回一个指向堆前一个顶部的指针。

![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/heap.png)

```c
#include<assert.h>
#include<string.h>
#include<sys/types.h>
#include<unistd.h>

void *malloc2(size_t size){
	void *p = sbrk(0);
	void *request = sbrk(size);
	if(request == (void*)-1){
		return NULL;
	}else{
		assert(p ==request);
		return p;
	}
}

int main(){

	int *p = (int *)malloc2(4);
	*p = 10;
       	printf("addr: %p\nvalue:  %d \n", p, *p);
		
	return 0;
}	

```

