---
categories:
- Understand Linux Kernel
date: 2014-06-08T14:05:00
tags:
- linux
- kernel
title: The Per-CPU Page Frame Cache
---

## 简介

在[上一篇文章]({{< ref ulk_pf >}}),
我们介绍了kernel对物理页帧的管理机制,
在实际的应用场景中,kernel经常申请和释放单个的物理页,
这在之前的模型下,效率会比较低.

因此为了提升在这样的场景下的系统的性能,
每个zone中引入了cpu自己的页帧缓存( `per-cpu` `frame` `cache` ).
每个cpu会维护一些预先分配的单个页的缓存用于该cpu
申请单个的物理页.

## 物理页cache类型

实际上对于每个zone来说有两种cache类型:

- `hot` `cache` : 这些物理页中存放的内容很可能就在当前的cpu硬件cache中.
- `cold` `cache` : 这些物理页中的内容不在当前cpu硬件cache中.

这里有必要说下cpu对物理内存的方式,当cpu尝试访问一个物理地址,
它首先会查看硬件cache中是否已经存在该地址处的内容,
如果已经存在那么直接返回cache中的内容,反之,
它会将真实物理内存处的内容,按照`cache` `line` 的大小加载到
cache中,然后再将其内容返回.

可见如果程序刚刚申请了一块内存,然后便尝试对其进行写操作,
那么如果该部分物理页还在cache中,就减少了一次内存加载.

当然, `cold` `cache` 也有其作用,当访问的物理页被用于 `DMA`
那么在这种情况下,就没必要使用 `hot` `cache` 的那些物理页.

## 数据类型

在每个zone当中存在一个数组,数组的个数为cpu的个数,
而每个数组的元素为 `struct` `per_cpu_pageset` ,
其实它只是对 `struct` `per_cpu_pages` 的一个封装:

```
struct per_cpu_pages {
	int count;		/* 该链表中物理页的个数 */
	int high;		/* 当链表中的物理页个数超过该数值,会将部分页返还给zone buddy系统 */
	int batch;		/* 每次返还给buddy系统的物理页的个数 */

	/*
	 * 对于不同的migrate type有自己的物理页链表,
	 * 其中hot cache的物理页存放在链表的前半部,
	 * 而cold cache存在放链表的后半部.
	 */
	struct list_head lists[MIGRATE_PCPTYPES];
};
```

## 申请

一般来说申请的流程比较简单,首先找到该cpu对应的链表,
如果其中存在物理页,之后根据 `gfp_flag` 中是否有
`__GFP_COLD` 标记,从而选择 `cold/hot` `cache` .

那么如果当前链表为空呢?这时就需要从 `zone` `buddy` 系统中
拿一些物理页用于填充.
这部分的操作在函数 `rmqueue_bulk` 中:

```
static int rmqueue_bulk(struct zone *zone, unsigned int order,
			unsigned long count, struct list_head *list,
			int migratetype, int cold)
{
	int mt = migratetype, i;

	spin_lock(&zone->lock);
	for (i = 0; i < count; ++i) {
		struct page *page = __rmqueue(zone, order, migratetype);
		if (unlikely(page == NULL))
			break;

		if (likely(cold == 0))
			list_add(&page->lru, list);
		else
			list_add_tail(&page->lru, list);
		...
		set_freepage_migratetype(page, mt);
		list = &page->lru;
		...
	}
	...
	spin_unlock(&zone->lock);
	return i;
}
```

注意,这里的 `order` 为0,也是说每次申请一个物理页,
而 `count` 则为 `pcp->batch` . `__rmqueue`
内部的流程可以参见[上一篇文章的分配章节]({{< ref "ulk_pf#分配" >}})
.

## 释放

释放的流程和申请类似,首先依然是找到当前cpu对应的链表,
如果是 `cold cache` 则将物理页挂入链表尾部,
反之插入到链表开头:

```
pcp = &this_cpu_ptr(zone->pageset)->pcp;
if (cold)
	list_add_tail(&page->lru, &pcp->lists[migratetype]);
else
	list_add(&page->lru, &pcp->lists[migratetype]);
pcp->count++;
```

如果该链表中的物理页超过了设定的阈值( `pcp->high` ),
那么需要将部分页归还给 `zone` `buddy` 系统:

```
if (pcp->count >= pcp->high) {
	free_pcppages_bulk(zone, pcp->batch, pcp);
	pcp->count -= pcp->batch;
}
```

这里由于存在多个 `migrate` `type` ,故当归还的时候,
采用了平均归还的策略,给每个 `migrate` `type` 都归还物理页.

这里需要注意的是,给还的 `migrate` `type` 只有这几个:

```
MIGRATE_UNMOVABLE,
MIGRATE_RECLAIMABLE,
MIGRATE_MOVABLE,
```

假如总共有6个物理页需要归还,
那么 `MIGRATE_UNMOVABLE` 会得到1个物理页,
`MIGRATE_RECLAIMABLE` 会得到2个物理页,
而 `MIGRATE_MOVABLE` 会得到3个物理页.

当然,如果中间某个 `migrate` `type` 的链表为空,
则会跳过该类型,而下个类型会被多归还一个:

```
do {
	batch_free++;
	if (++migratetype == MIGRATE_PCPTYPES)
		migratetype = 0;
	list = &pcp->lists[migratetype];
} while (list_empty(list));
```

确定了归还的 `migrate` `type` 的链表( `list` )
以及归还的物理页个数( `batch_free` ),
下面就是将其返回给 `buddy` 系统了,
这里的操作同释放物理页类型
(详见[上一篇文章的释放章节]({{< ref "ulk_pf#释放" >}})).

```
do {
	int mt;	/* migratetype of the to-be-freed page */

	page = list_entry(list->prev, struct page, lru);
	/* must delete as __free_one_page list manipulates */
	list_del(&page->lru);
	mt = get_freepage_migratetype(page);
	/* MIGRATE_MOVABLE list may include MIGRATE_RESERVEs */
	__free_one_page(page, zone, 0, mt);
	trace_mm_page_pcpu_drain(page, 0, mt);
	...
} while (--to_free && --batch_free && !list_empty(list));
```

注意,这里是从链表尾部开始释放,也就是说优先释放 `cold` `cache` 的物理页.

FIN.
