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
/*  chardev.c: Creates a read-only char device that says how many times
 *  you've read from the dev file
 *
 *  Copyright (C) 2001 by Peter Jay Salzman
 *
 *  08/02/2006 - Updated by Rodrigo Rubira Branco <rodrigo@kernelhacking.com>
 */

/* Kernel Programming */
#define MODULE
#define LINUX
#define __KERNEL__

#if defined(CONFIG_MODVERSIONS) && ! defined(MODVERSIONS)
   #include <linux/modversions.h>
   #define MODVERSIONS
#endif
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <asm/uaccess.h>  /* for put_user */
#include <asm/errno.h>

/*  Prototypes - this would normally go in a .h file */
int init_module(void);
void cleanup_module(void);
static int device_open(struct inode *, struct file *);
static int device_release(struct inode *, struct file *);
static ssize_t device_read(struct file *, char *, size_t, loff_t *);
static ssize_t device_write(struct file *, const char *, size_t, loff_t *);

#define SUCCESS 0
#define DEVICE_NAME "chardev" /* Dev name as it appears in /proc/devices   */
#define BUF_LEN 80            /* Max length of the message from the device */


/* Global variables are declared as static, so are global within the file. */

static int Major;            /* Major number assigned to our device driver */
static int Device_Open = 0;  /* Is device open?  Used to prevent multiple
                                        access to the device */
static char msg[BUF_LEN];    /* The msg the device will give when asked    */
static char *msg_Ptr;

static struct file_operations fops = {
  .read = device_read,
  .write = device_write,
  .open = device_open,
  .release = device_release
};


/* Functions */

int init_module(void)
{
   Major = register_chrdev(0, DEVICE_NAME, &fops);

   if (Major < 0) {
     printk ("Registering the character device failed with %d\n", Major);
     return Major;
   }

   printk("<1>I was assigned major number %d.  To talk to\n", Major);
   printk("<1>the driver, create a dev file with\n");
   printk("'mknod /dev/hello c %d 0'.\n", Major);
   printk("<1>Try various minor numbers.  Try to cat and echo to\n");
   printk("the device file.\n");
   printk("<1>Remove the device file and module when done.\n");

   return 0;
}


void cleanup_module(void)
{
   /* Unregister the device */
   int ret = unregister_chrdev(Major, DEVICE_NAME);
   if (ret < 0) printk("Error in unregister_chrdev: %d\n", ret);
}


/* Methods */

/* Called when a process tries to open the device file, like
 * "cat /dev/mycharfile"
 */
static int device_open(struct inode *inode, struct file *file)
{
   static int counter = 0;
   if (Device_Open) return -EBUSY;

   Device_Open++;
   sprintf(msg,"I already told you %d times Hello world!\n", counter++);
   msg_Ptr = msg;
   MOD_INC_USE_COUNT;

   return SUCCESS;
}


/* Called when a process closes the device file */
static int device_release(struct inode *inode, struct file *file)
{
   Device_Open --;     /* We're now ready for our next caller */

   /* Decrement the usage count, or else once you opened the file, you'll
                    never get get rid of the module. */
   MOD_DEC_USE_COUNT;

   return 0;
}


/* Called when a process, which already opened the dev file, attempts to
   read from it.
*/
static ssize_t device_read(struct file *filp,
   char *buffer,    /* The buffer to fill with data */
   size_t length,   /* The length of the buffer     */
   loff_t *offset)  /* Our offset in the file       */
{
   /* Number of bytes actually written to the buffer */
   int bytes_read = 0;

   /* If we're at the end of the message, return 0 signifying end of file */
   if (*msg_Ptr == 0) return 0;

   /* Actually put the data into the buffer */
   while (length && *msg_Ptr)  {

        /* The buffer is in the user data segment, not the kernel segment;
         * assignment won't work.  We have to use put_user which copies data from
         * the kernel data segment to the user data segment. */
         put_user(*(msg_Ptr++), buffer++);

         length--;
         bytes_read++;
   }

   /* Most read functions return the number of bytes put into the buffer */
   return bytes_read;
}


/*  Called when a process writes to dev file: echo "hi" > /dev/hello */
static ssize_t device_write(struct file *filp,
   const char *buff,
   size_t len,
   loff_t *off)
{
   printk ("<1>Sorry, this operation isn't supported.\n");
   return -EINVAL;
}

MODULE_LICENSE("GPL");
```

