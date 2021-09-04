---
title: linux kernel memmory allocate
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---



理解Linux内核中分配内存常用的接口函数的使用方法和实现原理：写一个内核模块，使用alloc_page分配一个物理页面，然后输出该页面的物理地址，并输出该物理页面在内核的虚拟地址，然后把这个物理页面全部填充为0x55。

简单做个测试，获取页大小：

`pr_info("PAGE_SIZE: %ld", PAGE_SIZE);`

输出：

```
/mnt # insmod allocmm.ko 
[ 5408.191046] PAGE_SIZE: 4096
```

物理页面的方法这里有详细介绍：

[Getting Pages](http://books.gigatux.nl/mirror/kerneldevelopment/0672327201/ch11lev1sec3.html)

主要函数是`struct page * alloc_pages(unsigned int gfp_mask, unsigned int order)`

它分配2^order个连续的物理页面，并返回一个指向第一页页面结构的指针 。你可以使用`void * page_address(struct page *page)`将给定的页面转换为其逻辑地址。 如果你不需要实际的struct page（即物理页面信息表示方法），可以直接调用`unsigned long __get_free_pages(unsigned int gfp_mask, unsigned int order)`

这个函数的工作原理与alloc_pages()相同，只是它直接返回第一个请求页面的逻辑地址。因为这些页面是连续的，所以其他页面只是从第一个页面开始。? 逻辑地址和物理地址一样都是连续的吗？？？

#### 方法1 __get_free_page

```c
	char *kbuf = (char *)__get_free_page(GFP_ATOMIC);
	unsigned long addr = __get_free_page(GFP_ATOMIC); 
	if(!kbuf){
		pr_info(" __get_free_page failed\n");
		return 0;
	}	
	pr_info("addr: 0x%lx\n", addr);
	pr_info(" __get_free_page start address 0x%lx!\n",(unsigned long)kbuf);
	free_page((unsigned long)kbuf);
	free_page(addr);
```

输出结果

```
/mnt # insmod allocmm.ko 
[ 8222.388107] PAGE_SIZE: 4096
[ 8222.388421] addr: 0xc453a000
[ 8222.388606]  __get_free_page start address 0xc453b000!
```



#### 方法2 kmalloc

kmalloc函数与用户空间的malloc一族函数非常类似, 只不过它多了一个flags参数, kmalloc函数是一个简单的接口, 用它可以获取以字节为单位的一块内核内存.

如果你需要整个页, 那么前面讨论的页分配接口是更好的选择. 但是, 对大多数内核分配来说, kmalloc接口用的更多。

```c
	char *buf2 = kmalloc((size_t)PAGE_SIZE, GFP_ATOMIC);
	if(!buf2){
		pr_info("kmalloc failed\n");
	}
	pr_info(" kmalloc start addess 0x%lx", (unsigned long)buf2);
	kfree(buf2);c
```

输出如下：

```
/mnt # insmod allocmm.ko 
[ 9487.064063] PAGE_SIZE: 4096
[ 9487.064601] addr: 0xc47ba000
[ 9487.064914]  __get_free_page start address 0xc4527000!
[ 9487.066320]  kmalloc start addess 0xc5280000
```



#### 方法3 vmalloc

 **kmalloc()用于申请较小的、连续的物理内存**

- 以字节为单位进行分配，在<linux/slab.h>中
- void \*kmalloc(size_t size, int flags) 分配的内存物理地址上连续，虚拟地址上自然连续

**vmalloc()用于申请较大的内存空间，虚拟内存是连续的**

- 以字节为单位进行分配，在<linux/vmalloc.h>中
- void vmalloc(unsigned long size) 分配的内存虚拟地址上连续，物理地址不连续

**kmalloc和vmalloc是分配的是内核的内存,malloc分配的是用户的内存**



```c
	char *vm_buf = vmalloc(PAGE_SIZE);
	if(!vm_buf){
		pr_info("vmalloc failed...\n");
		return 0;
	}

	pr_info("vmalloc start addres 0x%lx\n", (unsigned long)vm_buf);
```



输出：

```
/mnt # insmod allocmm.ko 
[10391.079627] PAGE_SIZE: 4096
[10391.079981] addr: 0xc44de000
[10391.080273]  __get_free_page start address 0xc4546000!
[10391.080720]  kmalloc start addess 0xc5280000
[10391.081347]  kmalloc start addess 0xc4529a00
[10391.082522] vmalloc start addres 0xc6abb000
```



## Buddy System

### 算法原理

伙伴系统，专门用来分配以页为单位的大内存，且分配的内存大小必须是2的整数次幂。这里的幂次叫做 `order`，例如一页的大小是4K，order为1的块就是 `2^1 * 4K = 8K`。每次分配时都寻找对应order的块，如果没有，就将order更高的块分裂为2个order低的块。释放时，如果两个order低的块是分裂出来的，就将他们合并为更高order的块。

在Linux中，使用buddy system分配的底层API主要有 `get_free_pages` 和 `alloc_pages`，传入的参数都是order，还有一些flag位.

值得注意的是这样分配得到的虚拟地址和物理地址都是连续的，返回的地址可以使用 `virts_to_phys` 或者 `__pa` 宏转换为物理地址，实际操作也就是加上了一个偏移而已。

可以通过 `/proc/buddyinfo` 和 `/proc/pagetypeinfo` 来查看相关的情况.

