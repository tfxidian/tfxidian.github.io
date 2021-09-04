---
title: linux kernel memmory allocate
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---





了解和熟悉使用slab机制分配内存，并理解slab机制的原理。

- 创建名为mycache的slab描述符，大小为20字节，align为8字节，flags为0，然后从这个slab描述符中分配一个空闲对象。
- 查看系统当前所有的slab



对于内核堆来说，只需要了解分配**大内存**的Buddy System和分配**小内存**的Slab。

slab分配器是一个抽象层，用于更容易地分配大量相同类型的对象。内核接口提供了函数kmem_cache_create，这个函数创建一个新的slab分配器，它将能够处理大小为size的对象分配。如果创建成功，将获得指向相关结构体kmem_cache的指针。这个结构保存着它所管理的slabs的信息。

![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/slab.png)

这张图可以说是介绍slab的文章中出现频次最高的了，我们只要记住，`kmem_cache`是类似于glibc arena的结构，每个`kmem_cache`由若干个slab构成，**每个slab由一个或多个连续的页组成**。`kmem_cache`有一个重要的性质，就是其中所有的object大小都是相同的（准确的说是分配块的大小都相同）.

```c
struct slab {
    struct list_head list; /* embedded list structure */
    unsigned long colouroff;
    void *s_mem; /* first object in the slab */
    unsigned int inuse; /* allocated objects in the slab */
    kmem_bufctl_t free; /* first free object (if any) */
};
```



相关API：

```c
struct kmem_cache * kmem_cache_create (	const char *name,
 	size_t  	size,
 	size_t  	align,
 	unsigned long  	flags,
 	void (*ctor(void*, struct kmem_cache *, unsigned long),
 	void (*dtor(void*, struct kmem_cache *, unsigned long));
```

创建mem_cache，需要指定name和size.

```c
void * kmem_cache_alloc (struct kmem_cache * cachep, gfp_t flags);
```

在mem_cache中分配object，这里不需要指定size因为在创建时就已经指定好了.

```C
void kmem_cache_free (struct kmem_cache * cachep, void * objp);
```

在mem_cache中释放object.

```c
void * kmalloc (size_t size, gfp_t flags);
```



代码如下：

```
	my_cache = kmem_cache_create("mycache", size, 0, SLAB_HWCACHE_ALIGN,NULL);
	if(!my_cache){
		pr_info("kmem_cache_create failed.\n");
		return 0;
	}
	pr_info("kmem_cache_create ok\n");
	kbuf = kmem_cache_alloc(my_cache, GFP_ATOMIC);
	if(!kbuf){
		pr_info("kmem_cache_alloc failed\n");
		return 0;
	}
	pr_info("kmem_cache_alloc ok \n");

	kmem_cache_free(my_cache, kbuf);
	kmem_cache_destroy(my_cache);
	pr_info("destroyed my_cache\n");

```



结果输出：

```
/mnt # insmod slab_lab.ko 
[ 5569.341997] kmem_cache_create ok
[ 5569.352925] kmem_cache_alloc ok 
[ 5569.391877] destroyed my_cache
```

