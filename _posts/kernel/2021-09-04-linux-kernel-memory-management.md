---
title: linux kernel memmory management
date: 2021-09-05 0:33:50
tags: linux, kernel
layout: post
---





# Slab、Buddy与二级分配

所有的内存肯定都是从buddy算法来的，buddy是最直面最底层的内存。也就是说无论谁通过何种方式申请内存，底层都会调用get_free_page()、page_alloc() 等buddy级的API.
 但buddy的粒度实在太大，每次分配的都至少是1页的4KB内存。 **那么APP只申请16个字节，那么4kb-16字节岂不是都废掉了？**
 所以linux申请内存是一个二级分配的过程。
 拿到2^n个内存页(4k)，应该如何去管理

![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/libc_buddy_slab.PNG)

这张图是宋宝华老师的PPT里面的一张截图，感觉看了这张图就已经解决了我心中的不少疑惑。之前只知道在访问的时候虚拟地址经过mmu访问物理地址。到底从用户态层面的申请的内存是怎么跟内核申请的串起来的呢？

- 内核空间申请通过kmalloc/kfree 都是**通过slab**去申请
- 用户层的 malloc 和 free 是**通过libc**去申请内存，libc底层通过brk ,sbrk, mmap 去管理内存，brk是进行堆扩展，mmap进行匿名映射。
- 都是一个二级管理，底层一定是向buddy 索要内存。

工作原理：

- 申请32b内存，那么slab向buddy 拿2^0 1页内存
- 将这一页切割成n 个 32b 大小的块
- 32b 的 slab 是一类slab(cache), 具有identical objects（等分）
  - 不会产生重叠，即32b slab中，会有64b的一块内存
- 用完了，再从buddy 中 要2^n 个page
- 其实就是 identical objects 瓜分 2^n 个 page



## libc 与 buddy

参照上面二级分配的原理图。malloc 与 free 都是通过libc 与buddy 进行打交道。



```c
 //libc 中有一个API mallopt, 参数 M_TRIM_THRESHOLD
 //应用程序在free内存的时候 free到多少的时候，libc才会把内存还给buddy
//设置位-1 最大的正整数，无论我free多少内存，内存都hold在libc中，不会还给buddy
int main(int argc, char ** argv)
{
  unsigned char * buffer;
  int i;
  if(!mlockall(MCL_CURRENT | MCL_FUTURE ))
    mallopt( M_TRIM_THRESHOLD, -1UL);
  mallopt(M_MMAP_MAX, 0);
  
  buffer = malloc(SOMESIZE);
  if( !buffer )
    exit(-1);

  /*
  * Touch each page in this piece of memory to get it
  *  mapped into RAM
  */  
  for( i = 0; i < SOMESIZE; i+= 4*1024)
      buffer[i] = 0;

  free(buffer);
  while(1);
  return 0;
}
```

上述代码适用于对实时性要求很高的场景。

申请一块很大的内存，写一次（后续再提），释放掉

并没有还给buddy, 而是被libc hold住了

