---
title: Linux kernel module learning tutorial(2)
date: 2021-08-11 23:55:50
tags: Linux, kernel module
layout: post
---

## 预备知识

### 模块是怎么开始和结束的
程序通常由main函数开始，历经一系列指令终止。内核模块的各种方式不太一样，一个模块通常由init_module或者你在module_init中制定的函数开始，这是内核模块的入口。它告诉内核这个模块提供了什么功能，并设置内核在需要时运行模块的函数


### 设备驱动
模块的一类是设备驱动程序，它为硬件(如串行端口)提供功能。在Unix上，每一块硬件都由位于/dev中的一个名为设备文件的文件表示，该文件提供了与硬件通信的方法。设备驱动程序代表用户程序提供通信。

设备分为两类:字符设备和块设备。不同之处在于块设备有一个请求缓冲区，因此它们可以选择响应请求的最佳顺序。这在存储设备的情况下很重要，在存储设备中，读或写彼此靠近的扇区比读或写那些相距较远的扇区更快。另一个区别是块设备只能以块的形式接收输入和返回输出(块的大小可以根据设备的不同而变化)，而字符设备可以使用任意多或少的字节。世界上大多数设备都是字符化的，因为它们不需要这种类型的缓冲，而且它们也不使用固定的块大小。
通过查看ls -l输出中的第一个字符，可以判断设备文件是用于块设备还是用于字符设备。如果是' b '，那么它是块设备，如果是' c '，那么它是字符设备。

## Character Device drivers
### The file_operations Structure
file_operations结构在include/linux/fs.h中定义，并保存着驱动程序定义的函数的指针，这些函数在设备上执行各种操作。结构的每个字段对应于驱动程序为处理请求的操作而定义的某个函数的地址。
例如，每个字符驱动程序都需要定义一个从设备读取的函数。file_operations结构保存了执行该操作的模块函数的地址
```
    struct file_operations {
       struct module *owner;
       loff_t (*llseek) (struct file *, loff_t, int);
       ssize_t (*read) (struct file *, char *, size_t, loff_t *);
       ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
       int (*readdir) (struct file *, void *, filldir_t);
       unsigned int (*poll) (struct file *, struct poll_table_struct *);
       int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
       int (*mmap) (struct file *, struct vm_area_struct *);
       int (*open) (struct inode *, struct file *);
       int (*flush) (struct file *);
       int (*release) (struct inode *, struct file *);
       int (*fsync) (struct file *, struct dentry *, int datasync);
       int (*fasync) (int, struct file *, int);
       int (*lock) (struct file *, int, struct file_lock *);
    	 ssize_t (*readv) (struct file *, const struct iovec *, unsigned long,
          loff_t *);
    	 ssize_t (*writev) (struct file *, const struct iovec *, unsigned long,
          loff_t *);
    };
```
有些操作不是由驱动程序实现的。例如，处理显卡的驱动程序不需要从目录结构中读取数据。有一个gcc扩展可以更方便地对这个结构进行赋值。你会在现代驱动上看到它，可能会让你大吃一惊。这是给结构赋值的新方法:
```
    struct file_operations fops = {
       read: device_read,
       write: device_write,
       open: device_open,
       release: device_release
    };
```

然而，也有一种C99的方式来分配结构的元素，这显然比使用GNU扩展更受欢迎。gcc支持新的C99语法。你应该使用这个语法，以防有人想要移植你的驱动程序。这将有助于兼容性:
```
    struct file_operations fops = {
       .read = device_read,
       .write = device_write,
       .open = device_open,
       .release = device_release
    };
```

直接上完整的例子

```
#include<linux/kernel.h>
#include<linux/module.h>
#include<linux/fs.h>
#include<asm/uaccess.h>
#include<asm/errno.h>
#define DEVICE_NAME "chardev"
#define BUF_LEN 80
#define MODULE
#define LINUX
#define __KERNEL__

int init_module(void);
void cleanup_module(void);

static int device_open(struct inode*, struct file*);
static int device_release(struct inode*, struct file*);

static ssize_t device_read(struct file*, char*, size_t, loff_t*);
static ssize_t device_write(struct file*, const char*, size_t, loff_t*);

static int Major;
static int Device_Open = 0;

static char msg[BUF_LEN];
static char *msg_Ptr;

static struct file_operations fops ={
	.read = device_read,
	.write = device_write,
	.open = device_open,
	.release = device_release,
};

int init_module(void)
{
	Major = register_chrdev(0, DEVICE_NAME, &fops);
	if(Major < 0){
		printk("Registering the character device failed with %d", Major);
	}

	printk("<1> I was assigned major number %d.\n", Major);


}


void cleanup_module(void)
{
	pr_info("this is cleanup in chardev.\n");
	unregister_chrdev(Major, DEVICE_NAME);
	//if(ret<0) printk("Error in unregister chardev. %d", ret);
	}

static int device_open(struct inode *inode, struct file *file)
{
	static int counter = 0;
	if(Device_Open) return -EBUSY;

	Device_Open++;
	sprintf(msg,"I already told you %d times hello\n", Device_Open);
	msg_Ptr = msg;
	return 0;
}

static int device_release(struct inode *inode, struct file*file){

	Device_Open --;
//	MOD_DEC_USE_COUNT;
	return 0;
}

static ssize_t device_read(struct file* filep, char* buffer, size_t length, loff_t* offset)
{
	if(*msg_Ptr ==0) return 0;
	int bytes_read = 0;
	while(length && *msg_Ptr){
		put_user(*(msg_Ptr++), buffer++);
		length--;
		bytes_read++;
	}
	return bytes_read;
}


static ssize_t device_write(struct file *filep, const char *buffer, size_t len, loff_t *off)
{
	printk("sorry this operation is not supported now.\n");
	return -EINVAL;
}


MODULE_LICENSE("GPL");

```

注意：如果将主设备号0传递给register_chrdev，则返回值将是动态分配的主设备号

中间遇到的问题：
在编译字符设备驱动文件时出现了一个 error: void value not ignored as it ought to be 错误。问题出在：
 `int ret = unregister_chrdev(Major,DEVICE_NAME); `

 改为：
 ```
 unregister_chrdev ( Major, DEVICE_NAME );
 ```

 下面是输出结果：
 ![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/chardev.PNG)

