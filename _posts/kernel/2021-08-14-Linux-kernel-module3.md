---
title: Linux kernel module learning tutorial(3)
date: 2021-08-14 16:15:50
tags: Linux, kernel module
layout: post
---


 ## The /proc File System
 proc文件系统包含一个虚拟的文件系统。它在磁盘上不存在。相反，内核在内存中创建它。它用于提供关于系统的信息(最初是关于进程的，因此得名)。
 ```
proc/1
一个包含进程号1的目录。每个进程在/proc下面都有一个目录，目录名是进程标识号。
/proc/cpuinfo
关于处理器的信息，例如它的类型、制造、型号和性能。
/proc/devices
配置到当前运行内核中的设备驱动程序列表。
/proc/dma
显示目前正在使用的DMA信道。
/proc/filesystems
配置到内核中的文件系统。
 ```

这里有一个简单的例子，展示了如何使用/proc文件。
这算是/proc文件系统的HelloWorld。
它由三部分组成:在函数init_module中创建文件/proc/helloworld，在回调函数procfile_read中读取文件/proc/helloworld时返回一个值(和一个缓冲区)，在函数cleanup_module中删除文件/proc/helloworld。
当使用proc_create函数加载模块时，会创建/proc/helloworld，返回值是一个结构体proc_dir_entry，它将用于配置文件/proc/helloworld(例如，该文件的所有者)，null返回值意味着创建失败


```
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

static int hello_proc_show(struct seq_file *m, void *v) {
  seq_printf(m, "Hello proc!\n");
  return 0;
}

static int hello_proc_open(struct inode *inode, struct  file *file) {
  return single_open(file, hello_proc_show, NULL);
}

static const struct file_operations hello_proc_fops = {
  .owner = THIS_MODULE,
  .open = hello_proc_open,
  .read = seq_read,
  .llseek = seq_lseek,
  .release = single_release,
};

static int __init hello_proc_init(void) {
  proc_create("hello_proc", 0, NULL, &hello_proc_fops);
  return 0;
}

static void __exit hello_proc_exit(void) {
  remove_proc_entry("hello_proc", NULL);
}

MODULE_LICENSE("GPL");
module_init(hello_proc_init);
module_exit(hello_proc_exit);
```

insmod myproc.ko之后，cat /proc/hello_proc,即可得到输出：
`Hello proc!`