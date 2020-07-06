---
categories:
- Understand Linux Kernel
date: 2014-02-23T16:00:00
tags:
- linux
- kernel
title: Permanent kernel mapping
---

## 简介

本文阐述一下linux kernel中permanent mapping的相关内容.
首先来说下,这个概念的背景,由于在32位的kernel地址空间中,
只有1G的可用空间(总共4G-用户空间3G),
那么如果实际的物理内存的大小超过1G,那么在kernel中,
将无法寻址超过1G的那部分物理内存(俗称高端内存),
为了解决这个问题,在kernel中引入了3个解决方案:

- permanent kernel mapping
- temporary kernel mapping
- noncontignous memory allocation

这里,我们只关注第一个(剩下2个等待后续分析),
它的主要思想是在内核1G的地址空间中,预留一部分空间,
专门用于映射那些物理地址超过1G的内存单元.

还有一点需要注意,顾名思义,该映射是永久的,
意思是一旦映射建立,除非是明确将其清除,
kernel并不会更改其映射.

知道了这些,下面来看下具体的实现细节.

## 初始化

初始化工作主要完成2件事:

- 确定kernel地址空间内用于该映射的地址范围
- 对上述地址范围对应的页表项进行初始化

该地址空间到底有多大呢?
其实,在kernel中有两个宏来确定的:

- LAST_PKMAP: 指定空间大小,单位是页的个数,其值根据PAE是否开启而决定:

```
#ifdef CONFIG_X86_PAE
#define LAST_PKMAP 512
#else
#define LAST_PKMAP 1024
#endif
```

- PKMAP_BASE: 指定空间的起始地址,其值为:

```
#define PKMAP_BASE ((FIXADDR_BOOT_START - PAGE_SIZE * (LAST_PKMAP + 1))	\
		    & PMD_MASK)
```

这里FIXADDR_TOP大家可以先理解成4G空间的顶端,
讲过上述的计算,如果我们假定一个页大小为4K,
可以看出当PAE开启时, 该空间大小为2M(4K * 512),
而当PAE禁止时,该空间大小为4M(4K * 1024),
而且该空间位于4GB空间的顶端.

知道了该空间的位置和大小,
剩下的工作便是初始这段空间对应的页表项,
这里会首先在低地址空间内申请映射该空间所需的物理页表.

```
unsigned long count = page_table_range_init_count(start, end);
```

之后遍历整个空间,初始化对应的页目录项,页中间目录项:

1. 找到该空间地址在页目录(swapper_pg_dir)中的表项,如果该表项为空,申请一块新的物理页当做页中间目录,并将其地址回填到页目录项中.

```
pmd = one_md_table_init(pgd);
```

2. 从获得的页中间目录(pmd)中获得该地址对应的表项,如果该表项不为空,将其内容复制到新申请的页表中,并将新页表的地址回填到页中间目录项中.

```
pte = page_table_kmap_check(one_page_table_init(pmd),
			    pmd, vaddr, pte, &adr);
```

最后将该空间起始地址对应的页表保存到一个全局变量中(pkmap_page_table),
为后续的申请和是否做准备.

```
pkmap_page_table = pte;
```

到这里,所有的初始化工作完成了,具体的代码可以参见init_32.c中的peramnent_kmaps_init函数.

## 申请

对于该空间的每个页表项都有个引用计数,用于标识该页的使用情况:

- 引用计数为0: 对应的页表项还没有被用于建立映射.
- 引用计数为1: 对应的页表项还没有被用于建立映射,但是还不能被使用,因为相对于的TLB还没有被刷新.
- 引用计数为n(n > 1): 对应的页表项已经建立了映射,有n-1个使用者.

同时,为了记录在该地址空间内的地址与其映射的物理页的联系,
这里kernel使用了一个hansh table专门用于记录.

如果kernel如果要尝试建立映射,首先调用的kmap,
该函数首先判断该页是否属于低端内存,
如果是的话,直接范围对应的虚拟地址,
因为低端内存(<1G)与kernel地址空间的映射关系很早就对应好了,
而且还是一一对应的关系,
反之,尝试进行高端内存映射.

```
void *kmap(struct page *page)
{
	might_sleep();
	if (!PageHighMem(page))
		return page_address(page);
	return kmap_high(page);
}
```

这里首先尝试获得该物理页对应的虚拟地址,
如果之前该物理页已经经过一次映射,
那么便直接返回之前建立的映射的虚拟地址.
反之说明是第一次建立映射.

```
vaddr = (unsigned long)page_address(page);
if (!vaddr)
	vaddr = map_new_virtual(page);
```

当然,最后将对应的页表项的引用计数+1:

```
pkmap_count[PKMAP_NR(vaddr)]++;
```

好了,下面来看第一次建立映射的情形.
这里有一个全局变量标识上一次建立映射的页表项的index,
我们尝试获得下一个页表项,如果上一次的index正好是最后一个页表项,
说明已经没有空闲的页表项,
那么我们需要对整个页表项进行刷新,
尝试释放一些没有建立映射的表项
(刷新对应的TLB).

```
for (;;) {
	last_pkmap_nr = (last_pkmap_nr + 1) & LAST_PKMAP_MASK;
	if (!last_pkmap_nr) {
		flush_all_zero_pkmaps();
		count = LAST_PKMAP;
	}
```

如果新的表项对应的引用计数为0,
说明可以直接使用.

```
if (!pkmap_count[last_pkmap_nr])
	break;	/* Found a usable entry */
```

如果引用计数不为0,那么我们继续往前遍历,
主要这里是从后往前遍历的.
如果所有的表项都被使用,
这时,我们只能等待其他使用者释放了,
这里讲当前线程挂入到一个全局的等待队列中,
当该线程被再次唤醒时,
重新检查该物理页是否在睡眠过程中被其他线程建立映射,
如果是的话,那么直接返回用于映射的地址,
否则的话,重新检查所有的页表项.

```
/*
 * Sleep for somebody else to unmap their entries
 */
{
	DECLARE_WAITQUEUE(wait, current);

	__set_current_state(TASK_UNINTERRUPTIBLE);
	add_wait_queue(&pkmap_map_wait, &wait);
	unlock_kmap();
	schedule();
	remove_wait_queue(&pkmap_map_wait, &wait);
	lock_kmap();

	/* Somebody else might have mapped it while we slept */
	if (page_address(page))
		return (unsigned long)page_address(page);

	/* Re-start */
	goto start;
}
```

经过上述的过程,我们现在获得了一个可用的页表项,
下面我们先获得对应的线性地址以及对应的页表项,
之后将高端内存的物理地址写入到对应的页表项中.

```
vaddr = PKMAP_ADDR(last_pkmap_nr);
set_pte_at(&init_mm, vaddr,
	   &(pkmap_page_table[last_pkmap_nr]), mk_pte(page, kmap_prot));
```

别忘了,最后将对应的页表项的引用计数置为1,
同时在全局的hash table中插入对应虚拟地址和物理页表的关系.

```
pkmap_count[last_pkmap_nr] = 1;
set_page_address(page, (void *)vaddr);
```

## 释放

释放的过程与申请的过程对应,调用kunmap,
如果释放的是低端内存,那么什么事也不做.

```
void kunmap(struct page *page)
{
	if (in_interrupt())
		BUG();
	if (!PageHighMem(page))
		return;
	kunmap_high(page);
}
```

反之,那么需要进行真正的释放.
首先找到对应的页表项:
在hash table中找到要释放的物理页对应的虚拟地址,
再根据得到的虚拟地址计算出页表项的index.

```
vaddr = (unsigned long)page_address(page);
BUG_ON(!vaddr);
nr = PKMAP_NR(vaddr);
```

得到了对应的页表项,那么便是将其引用计数-1,
如果结果为1,说明我们是该页表项的最后一个使用者,
这里,也许有另外一个线程正在等待一个可用的页表项,
那么我们便将其唤醒,反之,我们什么事都不做.

```
/*
 * A count must never go down to zero
 * without a TLB flush!
 */
need_wakeup = 0;
switch (--pkmap_count[nr]) {
case 0:
	BUG();
case 1:
	/*
	 * Avoid an unnecessary wake_up() function call.
	 * The common case is pkmap_count[] == 1, but
	 * no waiters.
	 * The tasks queued in the wait-queue are guarded
	 * by both the lock in the wait-queue-head and by
	 * the kmap_lock.  As the kmap_lock is held here,
	 * no need for the wait-queue-head's lock.  Simply
	 * test if the queue is empty.
	 */
	need_wakeup = waitqueue_active(&pkmap_map_wait);
}
unlock_kmap_any(flags);

/* do wake-up, if needed, race-free outside of the spin lock */
if (need_wakeup)
	wake_up(&pkmap_map_wait);
```

FIN.
