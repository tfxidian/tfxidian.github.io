---
title: Linux kernel module learning tutorial
date: 2021-08-03 23:55:50
tags: Linux kernel
layout: post
---

## basics of creating, compiling modules.

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

> Actually, things have changed starting with kernel 2.3.13, You can now use whatever name you like for the start and end functions of a module. In fact, the new method is the preferred method. However, many people still use init_module() and cleanup_module() for their start and end functions.



在之前目录下新建文件hello2.c

```
#include<linux/kernel.h>
#include<linux/module.h>
#include<linux/init.h>

static int __init hello2_init(void){
	pr_info("hello2 .\n");
	return 0;
}

static void __exit hello2_exit(void){

	pr_info("hello2 bye.\n");
}

module_init(hello2_init);
module_exit(hello2_exit);

MODULE_LICENSE("GPL");

```
使用了新的写法，用了自定义的init和exit函数。也包含了新的头文件<linux/init.h>。


## Passing Command Line Arguments to a Module

···
```
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/stat.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Peter Jay Salzman");

static short int myshort = 1;
static int myint = 420;
static long int mylong = 9999;
static char *mystring = "blah";

/*
 * module_param(foo, int, 0000)
 * The first param is the parameters name
 * The second param is it's data type
 * The final argument is the permissions bits,
 * for exposing parameters in sysfs (if non-zero) at a later stage.
 */

module_param(myshort, short, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP);
MODULE_PARM_DESC(myshort, "A short integer");
module_param(myint, int, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
MODULE_PARM_DESC(myint, "An integer");
module_param(mylong, long, S_IRUSR);
MODULE_PARM_DESC(mylong, "A long integer");
module_param(mystring, charp, 0000);
MODULE_PARM_DESC(mystring, "A character string");

static int __init hello_5_init(void)
{
        printk(KERN_ALERT "Hello, world 5\n=============\n");
        printk(KERN_ALERT "myshort is a short integer: %hd\n", myshort);
        printk(KERN_ALERT "myint is an integer: %d\n", myint);
        printk(KERN_ALERT "mylong is a long integer: %ld\n", mylong);
        printk(KERN_ALERT "mystring is a string: %s\n", mystring);
        return 0;
}

static void __exit hello_5_exit(void)
{
        printk(KERN_ALERT "Goodbye, world 5\n");
}

module_init(hello_5_init);
module_exit(hello_5_exit);
```

`insmod hello-5.o mystring="bebop" mybyte=255 myintArray=-1`