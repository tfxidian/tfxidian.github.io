---
title: Linux kernel module learning tutorial
date: 2021-08-03 23:55:50
tags: Linux kernel
layout: post
---

## basics of creating, compiling, installing and removing modules.

```

#include<linux/kernel.h>
#include<linux/module.h>

int init_module(void){
	pr_info("hello module.\n");
	return 0;
}

void cleanup_module(void){
	pr_info("hello end.\n");
}

MODULE_LICENSE("GPL");

```

Makefile

```
obj-m += hello.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

```

然后，执行make即可。
