---
categories:
- Understand Linux Kernel
date: 2014-03-23T15:24:00
tags:
- linux
- kernel
title: Noncontignous memory allocation
---

## 简介

之前我们已经说过了kernel对高端内存的2种使用方式,
分别为[永久映射]({{< ref ulk_pkmap >}})和[临时映射]({{< ref ulk_tkmap >}}),
今天,我们来说说最后一种方式: 非连续映射.

## 地址范围

在开始讲解分配和释放的过程之前,
我们有必要先搞清楚,非连续映射使用的线性地址范围.

![vmalloc_scope](vmalloc_scope.png)

- 从kernel线性地址空间的开头(PAGE_OFFSET)的896M和物理内存直接映射.
- 地址空间的结尾处用于建立固定映射,这其中包括我们说过的临时映射.
- 紧接着固定映射的下方,从PKMAP_BASE开始,用于建立永久映射.
- 那么剩下的地址空间就是用于建立非连续映射.需要注意的是,这里有一些用于安全考虑的空洞:首先是从直接映射结尾处到真正用于建立非连续映射的8M(VMALLOC_OFFSET)空洞,以及结尾的8M空洞,然后就是每个非连续映射的地址区域后面都有个4k的空洞.如果尝试访问这些空洞,会产生缺页异常,因为它们在页表中是不建立映射的.

最终,可用于建立非连续映射的地址范围由VMALLOC_START和VMALLOC_END指定:

32bit:

```
#define VMALLOC_START	((unsigned long)high_memory + VMALLOC_OFFSET)
#define VMALLOC_END	(PKMAP_BASE - 2 * PAGE_SIZE)
```

64bit:

```
#define VMALLOC_START    _AC(0xffffc90000000000, UL)
#define VMALLOC_END      _AC(0xffffe8ffffffffff, UL)
```

## 申请

对于非连续映射的申请主要分两部分:

1. 获得可用的非连续映射的线性地址空间.
2. 搜集可用的物理页,并将其与得到的线性地址建立映射.

我们先来看第一步,在kernel中,每个非连续地址空间都有个数据结构用来描述它:

```
struct vm_struct {
	struct vm_struct	*next;		// 指向下一个vm_struct
	void			*addr;			// 该区域开始的线性地址
	unsigned long		size;		// 该区域的大小+4096字节
	unsigned long		flags;		// 映射的区域的类型
	struct page		**pages;		// 指向对应的物理页数组
	unsigned int		nr_pages;	// 物理页数组的大小
	phys_addr_t		phys_addr;		// 当用于映射的区域类型为io,则指向对应的物理地址,反之则为0
	const void		*caller;		// vmalloc调用者的函数地址,调试用
};
```

可以看出这里所有的vm_struct使用单向链表连起来的,
出于对性能优化的考虑,kernel使用了红黑树来真正组织所有的非连续映射区域.

```
struct vmap_area {
	unsigned long va_start;			// 该区域的起始地址
	unsigned long va_end;			// 该区域的结束地址
	unsigned long flags;			// 该区域的类型
	struct rb_node rb_node;         // 在红黑树的位置
	struct list_head list;          // 在顺序链表中的位置
	struct list_head purge_list;    // 用于lazy purge的链表
	struct vm_struct *vm;			// 指向所属的vm_struct
	struct rcu_head rcu_head;		// rcu相关
};
```

下面我们主要来看下添加一个vmap_area的逻辑.

首先是查找对应的区域所在的位置,
这里从红黑树的根节点开始查找:

```
n = vmap_area_root.rb_node;
first = NULL;

while (n) {
	struct vmap_area *tmp;
	tmp = rb_entry(n, struct vmap_area, rb_node);
	if (tmp->va_end >= addr) {
		first = tmp;
		if (tmp->va_start <= addr)
			break;
		n = n->rb_left;
	} else
		n = n->rb_right;
}

if (!first)
	goto found;
```

最终,first指向第一个vmap_area,他的起始地址小于我们请求的地址.
如果最终其为空,说明是第一次请求,因为每次请求的开始地址都是VMALLOC_START.

下面我们就必须确定分配的区域范围的真实起始地址,因为现在的起始地址还是VMALLOC_START.
首先可以肯定的是该地址为至少应该为first->va_end,然后顺序向后遍历,
只到找到一个剩余空间足够大的vmap_area

```
/* from the starting point, walk areas until a suitable hole is found */
while (addr + size > first->va_start && addr + size <= vend) {
```

```
	addr = ALIGN(first->va_end, align);
	if (addr + size - 1 < addr)
		goto overflow;

	if (list_is_last(&first->list, &vmap_area_list))
		goto found;

	first = list_entry(first->list.next,
			struct vmap_area, list);
}
```

最终,初始化对应的vmap_area,并将其插入到红黑树以及一个顺序链表中:

```
va->va_start = addr;
va->va_end = addr + size;
va->flags = 0;
__insert_vmap_area(va);
```

找了空闲的线性空间,下面就是找到空间的物理页了进行映射了,
这里的过程比较简单,主要还是通过`alloc_page`接口去获得一个个的物理页,
最终在kernel页表中初始化对应的页目录项,页中间目录项,以及页表项.
这个过程就不再赘述,有兴趣的读者可以参见`__vmalloc_area_node`函数.

## 释放

释放的过程与申请相似,首先通过对应的线性地址,在红黑树中找到对应的vmap_area结构,
然后释放其对应的线性空间,这里为了性能的考量,并没有立刻释放,而是使用lazy_purge
技术,当需要释放的资源达到一定数量时,才一次性释放.

释放完了线性地址,下面就是释放对应的物理页了,依次遍历对应的物理页,
调用`free_page`将其解除映射.

释放的具体过程,有兴趣的读者可以参见`__vunmap`函数.

## 优化

通过上述的申请和释放的过程,大家可能发现这里还是有优化的空间:

- 每次查找可用的线性地址区域,没必要从VMALLOC_START开始.

的确,如果能找到一个节点,他的剩余可用空间足够大的话,完全可以从它的va_end开始查找.
kernel中也的确采用了类似的优化,首先是在申请时,如果存在可用的节点,
直接使用其va_end作为起始地址进行下一步的查找:

```
/* find starting point for our search */
if (free_vmap_cache) {
	first = rb_entry(free_vmap_cache, struct vmap_area, rb_node);
	addr = ALIGN(first->va_end, align);
	if (addr < vstart)
		goto nocache;
	if (addr + size - 1 < addr)
		goto overflow;
```

每次申请新的vmap_area时,free_vmap_cache都会新分配的vmap_area,
同样在释放vmap_area时,如果释放的空间区域的起始地址小于free_vmap_cache,
其同样也会更新

```
struct vmap_area *cache;
cache = rb_entry(free_vmap_cache, struct vmap_area, rb_node);
if (va->va_start <= cache->va_start) {
	free_vmap_cache = rb_prev(&va->rb_node);
```

当然,是否选择free_vmap_cache也是有条件的,先来看几个全局变量:

```
static unsigned long cached_hole_size; // 在free_vmap_cache->va_start地址下面的最大的空闲区域的大小
static unsigned long cached_vstart; // 上一次申请的区域的起始地址
static unsigned long cached_align; // 上一次申请的区域的对其大小
```

在申请时,当满足这些条件时,free_vmap_cache被认为是invalid的:

```
if (!free_vmap_cache ||
		size < cached_hole_size ||
		vstart < cached_vstart ||
		align < cached_align) {
nocache:
	cached_hole_size = 0;
	free_vmap_cache = NULL;
}
```

可以看出,读者可能会有一个疑问,为什么申请大小小于空闲区域的大小时
不使用free_vmap_cache呢?大家可以反过来想,如果申请的大小大于空闲大小,
那么可用的空间地址一定是从free_vmap_cache->va_end开始,因为之前的都不满足.

同理在释放时,如果释放的区域结束地址小于上一次申请的区域的开始地址,
free_vmap_cache同样会被置为空:

```
if (va->va_end < cached_vstart) {
	free_vmap_cache = NULL;
```

FIN.
