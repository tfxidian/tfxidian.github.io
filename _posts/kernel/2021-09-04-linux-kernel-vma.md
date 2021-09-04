---
title: linux kernel memmory vma
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---





了解和熟悉使用slab机制分配内存，并理解slab机制的原理。



遍历用户进程的所有VMA，并且打印VMA的属性，比如大小、起始地址。

**tips:** 在Linux 内核中，current是一个指向当前进程的指针。

```
static __always_inline struct task_struct *get_current(void)
{
    return percpu_read_stable(current_task);
}

#define current get_current()
```



先写一个测试，打印当前（也可以接受输入）进程的简单信息：

```c
	struct task_struct *tsk;
	if(pid ==0){
		tsk = current;   //get current process task
		pid = current->pid;
	}else{
		tsk = pid_task(find_vpid(pid), PIDTYPE_PID);//get input pid task
	}
	if(!tsk) return -1;
	pr_info("tsk pid: %d, command :%s", pid, tsk->comm);
	printit(tsk);

```

输出信息如下：

```
/mnt # insmod vma_test.ko 
[11124.554062] tsk pid: 811, command :insmod
[11124.554384] printit...
```

遍历内存信息

```c
	struct mm_struct *mm;
	struct vm_area_struct *vma;
	mm = tsk->mm;
	pr_info("mm = %p\n", mm);
	vma = mm->mmap;
	unsigned long start, end, length;
	while(vma){
		start = vma->vm_start;
		end = vma->vm_end;
		length = end -start;
		pr_info("vma: %p, start: %x, end: %x, length: %ld\n", vma, start, end, length);
		vma = vma->vm_next;
	}
```

输出结果如下：

```
/mnt # insmod vma_test.ko 
[12061.078337] tsk pid: 827, command :insmod
[12061.078677] mm = c3c4c480
[12061.078921] vma: c3c58528, start: 10000, end: 207000, length: 2060288
[12061.079339] vma: c3c58a50, start: 216000, end: 219000, length: 12288
[12061.079626] vma: c3c58b58, start: 219000, end: 23d000, length: 147456
[12061.080105] vma: c3c58738, start: bed70000, end: bed92000, length: 139264
[12061.080595] vma: c3c597b8, start: bedd1000, end: bedd2000, length: 4096
```

