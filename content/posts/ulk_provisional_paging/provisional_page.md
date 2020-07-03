---
categories:
- learning
date: 2014-01-19T18:00:00
tags:
- linux
- kernel
- intel
title: Provisional Paging Setup in Linux
---

## 概述

本文主要讲解下,linux kernel在实模式(real mode)中是如何建立所谓的临时页表的,
有了这些临时页表,便可以开启cpu页管理单元, 进入真正的保护模式(protect mode).
这里只考虑32位x86平台.

## 评估

在开始建立页表之前,我们有必要知道,我们到底需要映射多大的内存.
这里需要考虑几方面的要求:

- 能够容纳整个kernel image.
- 能够放下所有的页表项.
- 不应该映射过大的空间,因为这些映射只是临时的.

之所以说是临时,因为现在我们还不知道真实的物理内存是多大,
当知道了真实的物理内存大小,我们需要重新建立合适的页表.

先来看kernel image有多大,这个信息我们可以通过链接脚本来获得:

```
SECTIONS
{

	...

	/* Text and read-only data */
	.text :  AT(ADDR(.text) - LOAD_OFFSET) {
		_text = .;
		...
		/* End of text section */
		_etext = .;
	} :text = 0x9090

	...

	/* Data */
	.data : AT(ADDR(.data) - LOAD_OFFSET) {
		/* Start of data section */
		_sdata = .;
		...
		/* End of data section */
		_edata = .;
	} :data

	...

	. = ALIGN(PAGE_SIZE);
	.brk : AT(ADDR(.brk) - LOAD_OFFSET) {
		__brk_base = .;
		. += 64 * 1024;		/* 64k alignment slop space */
		*(.brk_reservation)	/* areas brk users have reserved */
		__brk_limit = .;
	}

	_end = .;

    ...
}
```

这里有很多内容被省略了,我们来几个关键的导出符号:

- _text,_etext: kernel代码段的开始和结束地址.
- _sdata,_edata: kernel数据段的开始和结束地址.
- __brk_base,__brk_limit: kernel初始时堆的开始与结束地址,需要注意的是,kernel将所有的初始化好的页表都放在堆的开始处.

*NOTE:* 这里所说的地址都是指运行时的虚拟地址.

知道了image的大小,下面就是要计算所有页表项需要占用的空间.

```
/* Enough space to fit pagetables for the low memory linear map */
MAPPING_BEYOND_END = PAGE_TABLE_SIZE(LOWMEM_PAGES) << PAGE_SHIFT
```

由于kernel只能使用线性地址空间中从PAGE_OFFSET(0xc0000000)开始的1Gb(1<<32 -
0xc0000000)的空间,所以,我只需要计算这部分空间所占用的页表即可.

在代码中,这部分地址空间被称为LOWMEM,其所需的物理页的个数为:

```
/* Number of possible pages in the lowmem region */
LOWMEM_PAGES = (((1<<32) - __PAGE_OFFSET) >> PAGE_SHIFT)
```

下面需要考虑2中情况,一个是开启PAE功能,另一种是未开启的情形.

先来看PAE disable情形下,这种情形比较简答,因为是使用的2级分层模式,
所以只需要计算有几个页目录项即可,每个页目录项指向一个页表,
每个页表含有1024个页表项,每个页表项是对应物理页的物理地址,
所以,一个页表是4Kb,正好是一个物理页的大小,
那么我们只需要计算有几个页表即可:

```
#define PAGE_TABLE_SIZE(pages) ((pages) / PTRS_PER_PGD)
```

那么如果PAE enable呢,这时采用的是3级分层模式,
首先我们任然需要一个页目录项,不过这时它指向的是一个页中间目录(PMD),
该表含有512个表项,每个表项8个字节,指向一个页表,
所以该表页正好占一个物理页的大小,那么我们这里除了和之前一样计算需要多少页表,
还需要再加上一个页中间目录:

```
#define PAGE_TABLE_SIZE(pages) (((pages) / PTRS_PER_PMD) + PTRS_PER_PGD)
```

## 页表初始化

知道了我们需要映射的大小,下面就是具体的页表初始化了.
先来开PAE enable的情形, 其中页中间目录是存放kernel的BSS段中的:

```
initial_pg_pmd:
	.fill 1024*KPMDS,4,0
```

而页目录是存放在数据段中,其中有部分表项已被静态初始化(主要用于bios,不再本文讨论范围):

```
__PAGE_ALIGNED_DATA
	/* Page-aligned for the benefit of paravirt? */
	.align PAGE_SIZE
ENTRY(initial_page_table)
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR),0	/* low identity map */
# if KPMDS == 3
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR),0
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR+0x1000),0
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR+0x2000),0
# elif KPMDS == 2
	.long	0,0
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR),0
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR+0x1000),0
# elif KPMDS == 1
	.long	0,0
	.long	0,0
	.long	pa(initial_pg_pmd+PGD_IDENT_ATTR),0
# else
#  error "Kernel PMDs should be 1, 2 or 3"
# endif
	.align PAGE_SIZE		/* needs to be page-sized too */
```

剩下的页表都是放在预留的堆空间(从__brk_base开始):

```
	xorl %ebx,%ebx				/* 清0 ebx寄存器 */

	movl $pa(__brk_base), %edi /* edi = 页表开始的物理地址 */
	movl $pa(initial_pg_pmd), %edx /* edx = 页中间目录的开始地址 */
	movl $PTE_IDENT_ATTR, %eax /* eax = 表项flag(rw+present)*/
10:
	leal PDE_IDENT_ATTR(%edi),%ecx		/* ecx = 页表物理地址 | 表项flag */
	movl %ecx,(%edx)			/* 将组合的结果填入到页中间目录的第一个表项中 */
	addl $8,%edx /*指向一个页中间目录的表项 */
	movl $512,%ecx /* 设置循环计数为页目录中表项的个数(512) */
11:
	stosl /* eax = 物理地址 | 表项flag, 将组合的结果填入到对应的页表中的表项低32位 */
	xchgl %eax,%ebx /* eax = 0 */
	stosl /* 表项的高32位清0 */
	xchgl %eax,%ebx /* eax = 0 */
	addl $0x1000,%eax /* eax += 0x1000(一个物理页的大小) */
	loop 11b

	/*
	 * 循环结束的条件: 映射大小 >= end + MAPPING_BEYOND_END.
	 */
	movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp
	cmpl %ebp,%eax
	jb 10b
1:
	addl $__PAGE_OFFSET, %edi
	movl %edi, pa(_brk_end) /* _brk_end = 映射的最大地址 */
	shrl $12, %eax
	movl %eax, pa(max_pfn_mapped) /* max_pfn_mapped = 最大的映射的物理页号

	/* 建立fix map映射表(暂不讨论) */
	movl $pa(initial_pg_fixmap)+PDE_IDENT_ATTR,%eax
	movl %eax,pa(initial_pg_pmd+0x1000*KPMDS-8)
```

当PAE disable, 页目录是存放在BSS段中的:

```
ENTRY(initial_page_table)
	.fill 1024,4,0
```

页表初始化的过程和PAE enable时差不多:

```
	/*
	 * kernel开始的虚拟地址在页目录的偏移 = ((__PAGE_OFFSET >> 22) << 2)
	 * 因为每个表项大小为4个字节,所以需要左移2位
	 */
page_pde_offset = (__PAGE_OFFSET >> 20);

	movl $pa(__brk_base), %edi
	movl $pa(initial_page_table), %edx
	movl $PTE_IDENT_ATTR, %eax
10:
	leal PDE_IDENT_ATTR(%edi),%ecx		/* Create PDE entry */
	movl %ecx,(%edx)			/* Store identity PDE entry */
	movl %ecx,page_pde_offset(%edx)		/* Store kernel PDE entry */
	addl $4,%edx
	movl $1024, %ecx
11:
	stosl
	addl $0x1000,%eax
	loop 11b
	/*
	 * End condition: we must map up to the end + MAPPING_BEYOND_END.
	 */
	movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp
	cmpl %ebp,%eax
	jb 10b
	addl $__PAGE_OFFSET, %edi
	movl %edi, pa(_brk_end)
	shrl $12, %eax
	movl %eax, pa(max_pfn_mapped)

	/* Do early initialization of the fixmap area */
	movl $pa(initial_pg_fixmap)+PDE_IDENT_ATTR,%eax
	movl %eax,pa(initial_page_table+0xffc)
```

## 开启页管理单元

万事俱备,只欠东风,最后一步便是开始页管理单元,
这里需要做2件事:

- 将页目录的物理地址放入cr3寄存器.
- 将cr0中的PG标记置1.

代码中也的确是这么做的:

```
movl $pa(initial_page_table), %eax
movl %eax,%cr3		/* set the page table pointer.. */
movl $CR0_STATE,%eax
movl %eax,%cr0		/* ..and set paging (PG) bit */
```

FIN.
