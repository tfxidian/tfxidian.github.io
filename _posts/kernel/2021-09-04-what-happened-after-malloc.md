---
title: What happened after malloc
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---





我在一次面试中被问到这个问题。他们想知道的是，**当用户调用malloc()分配内存时，发生了什么？调用了哪些函数？操作系统是如何响应的？**



现在试着解答这个问题：

当用户空间应用程序调用malloc()时，该调用不会在内核中实现。相反，它是一个库调用(实现了glibc或类似的)。简单来说，glibc中的malloc实现要么从brk()/sbrk()系统调用中获取内存，要么通过mmap()获取匿名内存。这给glibc提供了一个大的连续(关于虚拟内存地址)内存块，malloc实现进一步将其切片并分割成更小的块并分发给应用程序。

注意，目前还不关心物理内存——当通过brk()/sbrk()或mmap()改变进程数据段时，以及当内存被引用(通过对内存的读或写)时，物理内存由内核虚拟内存系统处理。

下面是一个malloc分配器的简单实现：

```
/* Include the sbrk function */

#include <unistd.h> 

int has_initialized = 0;
void *managed_memory_start;
void *last_valid_address;

void malloc_init()
{ 
 /* grab the last valid address from the OS */  
 last_valid_address = sbrk(0);     

 /* we don't have any memory to manage yet, so 
  *just set the beginning to be last_valid_address 
  */  
 managed_memory_start = last_valid_address;     

 /* Okay, we're initialized and ready to go */
  has_initialized = 1;   
}

struct mem_control_block {
 int is_available;
 int size;
};

void free(void *firstbyte) {
 struct mem_control_block *mcb;

 /* Backup from the given pointer to find the
  * mem_control_block
  */
 mcb = firstbyte - sizeof(struct mem_control_block);
 /* Mark the block as being available */
 mcb->is_available = 1;
 /* That's It!  We're done. */
 return;
}

void *malloc(long numbytes) {
 /* Holds where we are looking in memory */
 void *current_location;

 /* This is the same as current_location, but cast to a
  * memory_control_block
  */
 struct mem_control_block *current_location_mcb;

 /* This is the memory location we will return.  It will
  * be set to 0 until we find something suitable
  */
 void *memory_location;

 /* Initialize if we haven't already done so */
 if(! has_initialized)  {
  malloc_init();
 }

 /* The memory we search for has to include the memory
  * control block, but the user of malloc doesn't need
  * to know this, so we'll just add it in for them.
  */
 numbytes = numbytes + sizeof(struct mem_control_block);

 /* Set memory_location to 0 until we find a suitable
  * location
  */
 memory_location = 0;

 /* Begin searching at the start of managed memory */
 current_location = managed_memory_start;

 /* Keep going until we have searched all allocated space */
 while(current_location != last_valid_address)
 { 
  /* current_location and current_location_mcb point
   * to the same address.  However, current_location_mcb
   * is of the correct type so we can use it as a struct.
   * current_location is a void pointer so we can use it
   * to calculate addresses.
   */
  current_location_mcb =
   (struct mem_control_block *)current_location;

  if(current_location_mcb->is_available)
  {
   if(current_location_mcb->size >= numbytes)
   {
    /* Woohoo!  We've found an open,
     * appropriately-size location.
     */

    /* It is no longer available */
    current_location_mcb->is_available = 0;

    /* We own it */
    memory_location = current_location;

    /* Leave the loop */
    break;
   }
  }

  /* If we made it here, it's because the Current memory
   * block not suitable, move to the next one
   */
  current_location = current_location +
   current_location_mcb->size;
 }

 /* If we still don't have a valid location, we'll
  * have to ask the operating system for more memory
  */
 if(! memory_location)
 {
  /* Move the program break numbytes further */
  sbrk(numbytes);

  /* The new memory will be where the last valid 
   * address left off 
   */
  memory_location = last_valid_address;

  /* We'll move the last valid address forward 
   * numbytes 
   */
  last_valid_address = last_valid_address + numbytes;

  /* We need to initialize the mem_control_block */
  current_location_mcb = memory_location;
  current_location_mcb->is_available = 0;
  current_location_mcb->size = numbytes;
 }

 /* Now, no matter what (well, except for error conditions),
  * memory_location has the address of the memory, including
  * the mem_control_block
  */

 /* Move the pointer past the mem_control_block */
 memory_location = memory_location + sizeof(struct mem_control_block);

 /* Return the pointer */
 return memory_location;
 }
```



总结:

- Malloc()将搜索它的内存池，以查看是否有一块未使用的内存满足分配需求。
- 如果失败，malloc()将尝试扩展进程数据段(通过sbrk()/brk()或在某些情况下mmap())，Sbrk()最终在内核中结束（实际上使用由buddy分配器分配的内存）。
- 内核中的brk()/sbrk()调用会调整进程的struct mm_struct中的一些偏移量，因此进程数据段会变大。首先，没有物理内存映射到扩展数据段所提供的额外虚拟地址。
- 未映射内存第一次被访问(很可能是malloc实现的读/写操作)时，错误处理程序将进入并捕获到内核，内核将在那里为未映射内存分配物理内存。

请注意：Malloc不直接处理物理内存。在面试的时候我想多了，想着怎么跟后面的物理内存分配连起来，实际上那是另一个问题了。