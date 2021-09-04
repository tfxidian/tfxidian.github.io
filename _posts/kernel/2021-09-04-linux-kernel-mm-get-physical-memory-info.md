---
title: linux kernel physics memory
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---

## Linux kernel之物理内存信息

- 实验2：获取系统的物理内存信息

  ![](https://github.com/tfxidian/tfxidian.github.io/raw/master/pic/mm_lab2.PNG)

#### 获取物理内存的页面数量

这个利用get_num_physpages()函数

```
static inline unsigned long get_num_physpages(void)
{
	int nid;
	unsigned long phys_pages = 0;

	for_each_online_node(nid)
		phys_pages += node_present_pages(nid);

	return phys_pages;
}
```



获取系统所有内存的大小，返回物理内存的页面数量

```c
long phy_nums = get_num_physpages();
	pr_info("phy_nums: %ld\n", phy_nums);
```

输出

```
/mnt # insmod mm_info.ko 
[   42.242842] phy_nums: 25600
```

获取物理内存起始页帧号，可以直接用ARCH_PFN_OFFSET来取值。

```cpp
#define PAGE_SHIFT		12
/* 物理内存起始地址 */
#define PHYS_OFFSET	((phys_addr_t)__pv_phys_pfn_offset << PAGE_SHIFT)
/* 物理内存起始页帧号 */
#define PHYS_PFN_OFFSET	(__pv_phys_pfn_offset)
#define ARCH_PFN_OFFSET		PHYS_PFN_OFFSET
```

```c
...
pr_info("ARCH_PFN_OFFSET: %ld", ARCH_PFN_OFFSET);
...
```

输出如下：

```
/mnt # insmod mm_info.ko 
[  712.878257] phy_nums: 25600
[  712.879590] ARCH_PFN_OFFSET: 393216
```

在得到物理内存起始页帧号后，就可以通过遍历得到所有物理内存页帧号，然后再利用pfn_to_page函数就可以得到该页的信息。对于每一个struct page *p，可以通过各种函数查看它的信息，包括是否有效，是否空闲，是否上锁等等，就不再一一列举。

代码段如下：

```c
	unsigned long i;
	unsigned long pfn = 0, valid_pfn = 0, free_pfn = 0;
	for(i = 0; i< phy_nums; i++){
		pfn = i + ARCH_PFN_OFFSET;
		if(!pfn_valid(pfn)){
			continue;
		}
		valid_pfn++;
		struct page* p = pfn_to_page(pfn);
		if(!p){
			continue;
		}
		if(!page_count(p)){
			free_pfn++;
			continue;
		}
	}

```

输出结果如下：

```
/mnt # insmod mm_info.ko 
[ 2080.237695] phy_nums: 25600
[ 2080.238705] ARCH_PFN_OFFSET: 393216
[ 2080.255665] valid_pfn: 25600 free pfn: 13412
```

