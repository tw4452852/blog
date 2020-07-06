---
categories:
- Understand Linux Kernel
date: 2014-07-24T22:00:00
tags:
- linux
- kernel
title: Memory Area Management - The Slob Allocator
---

## 简介

`slob` (Simple List Of Blocks)
是一个为嵌入式系统特殊定制的内存分配器,
通常来说这类系统只有比较少的内存可供使用.
`slob` 使用一种非常简单的优先匹配的算法来查找可用的memory area,
而不像古老的 `K&R-style heap allocator`.
`slob`非常适合对内存使用有严格要求的系统,
不过它也有它的缺点:内存碎片.

## Unit

在 `slob` 中所有的memory area的大小的计量单位被称为 `slobidx_t`,
我们先来看下它的定义:

```
#if PAGE_SIZE <= (32767 * 2)
typedef s16 slobidx_t;
#else
typedef s32 slobidx_t;
#endif
```

可以看出,根据物理页的大小,该最小单位可能是 `2/4` `bytes`.

而每个 `slob` 的描述结构就是由 `slobidx_t` 组成:

```
struct slob_block {
	slobidx_t units;
};
typedef struct slob_block slob_t;
```

## Slob management

所有的 `slob` 根据大小会被放入到3个全局链表中:

```
#define SLOB_BREAK1 256
#define SLOB_BREAK2 1024
static LIST_HEAD(free_slob_small);
static LIST_HEAD(free_slob_medium);
static LIST_HEAD(free_slob_large);
```

- 大小小于256字节的 `slob` 放入到 `free_slob_small`
- 大小小于1024字节的 `slob` 放入到 `free_slob_medium`
- 大小小于一个物理页的 `slob` 放入到 `free_slob_large`
- 大于一个物理页的话,直接从 `zone` `page` `allocator` 中分配

## Allocating a object from cache

有了上面的基本概念,我们就可以看看从一个 `cache` 中是如何分配一个 `object`.

`slob` 对外提供的分配接口为 `kmem_cache_alloc_node`:

```
void *kmem_cache_alloc_node(struct kmem_cache *c, gfp_t flags, int node)
```

如果 `object` 的大小小于一个物理页的大小,从 `slob` 中查找,
反之,直接从 `zone` `page` `allocator` 中查找.
当找到可用的memory area,对其进行初始化之后便可以返回给使用者:

```
if (c->size < PAGE_SIZE) {
	b = slob_alloc(c->size, flags, c->align, node);
	...
} else {
	b = slob_new_pages(flags, get_order(c->size), node);
	...
}

if (c->ctor)
	c->ctor(b);
```

其中 `slob_new_pages` 只是对 `allocate_pages` 的简单封装:

```
static void *slob_new_pages(gfp_t gfp, int order, int node)
{
	void *page;

#ifdef CONFIG_NUMA
	if (node != NUMA_NO_NODE)
		page = alloc_pages_exact_node(node, gfp, order);
	else
#endif
		page = alloc_pages(gfp, order);

	if (!page)
		return NULL;

	return page_address(page);
}
```

这里我们主要关注如何从 `slob` 中分配一个 `object`.

首先根据分配的大小,确定 `slob` 从哪里找:

```
if (size < SLOB_BREAK1)
	slob_list = &free_slob_small;
else if (size < SLOB_BREAK2)
	slob_list = &free_slob_medium;
else
	slob_list = &free_slob_large;
```

确定了查找的方向,下面就是遍历该链表上的 `slob`,
如果 `slob` 上的可用空间小于需要的分配的大小,
继续遍历该链表上的下一个 `slob`:

```
spin_lock_irqsave(&slob_lock, flags);
/* Iterate through each partially free page, try to find room */
list_for_each_entry(sp, slob_list, list) {
```

```
if (sp->units < SLOB_UNITS(size))
	continue;
```

这里的 `SLOB_UNITS` 宏是将以字节为单位的大小转换成以 `slobidx_t`
为单位,并做相应的对齐:

```
#define SLOB_UNIT sizeof(slob_t)
#define SLOB_UNITS(size) (((size) + SLOB_UNIT - 1)/SLOB_UNIT)
```

如果当前的 `slob` 满足要求,那么我们尝试从中获得可用的空间,
如果尝试失败,继续查找下一个 `slob`:

```
prev = sp->list.prev;
b = slob_page_alloc(sp, size, align);
if (!b)
	continue;
```

这里我们先跳过从一个 `slob` 中分配的流程,继续往下看.

如果找到了可用的memory,我们便可以退出遍历了.
不过,为了防止每次遍历都从链表开头遍历,
因为从链表开头到当前节点之前的所有 `slob` 的可用空间都不够,
为了防止每次重复的遍历,这里做了优化,将链表头 `slob_list`
放到当前节点 `prev->next` 的前面,这样,下次遍历时,
便从当前节点开始查找:

```
/* Improve fragment distribution and reduce our average
 * search time by starting our next search here. (see
 * Knuth vol 1, sec 2.5, pg 449) */
if (prev != slob_list->prev &&
		slob_list->next != prev->next)
	list_move_tail(slob_list, prev->next);
```

如果该链表上所有 `slob` 都不能满足要求,
那么我们只能申请一个新的 `slob` 了,
这里,我们新申请的 `slob` 的大小为一个物理页的大小.

```
if (!b) {
	b = slob_new_pages(gfp & ~__GFP_ZERO, 0, node);
	if (!b)
		return NULL;
	sp = virt_to_page(b);
	__SetPageSlab(sp);
```

下面就是对新申请的 `memory` `area` 进行初始化了:

```
sp->units = SLOB_UNITS(PAGE_SIZE);
sp->freelist = b;
INIT_LIST_HEAD(&sp->list);
set_slob(b, SLOB_UNITS(PAGE_SIZE), b + SLOB_UNITS(PAGE_SIZE));
set_slob_page_free(sp, slob_list);
```

需要注意的是, `slob` 的管理信息是存放在 `struct` `page`
结构体中的,而不像 `slab` 需要申请额外的空间存放这些信息.

- `sp->unites:` 该 `slob` 的大小
- `sp->freelist:` 该 `slob` 上可用的空闲的起始地址
- 初始化 `slob_t` 的大小,以及下一个 `slob_t` 与当前节点的偏移:

```
static void set_slob(slob_t *s, slobidx_t size, slob_t *next)
{
	slob_t *base = (slob_t *)((unsigned long)s & PAGE_MASK);
	slobidx_t offset = next - base;

	if (size > 1) {
		s[0].units = size;
		s[1].units = offset;
	} else
		s[0].units = -offset;
}
```

如果memory area的大小大于一个 `unit`,
第一个 `unit` 中存放该大小,
第二个 `unit` 中存放偏移
反之,说明大小只够容纳一个 `unit`,
那么该 `unit` 中存放偏移的负值.

最后将 `slob` 挂入到对应的链表中:

```
static void set_slob_page_free(struct page *sp, struct list_head *list)
{
	list_add(&sp->list, list);
	__SetPageSlobFree(sp);
}
```

最终,我们尝试从该 `slob` 分配一个 `object`.
这里我们依次遍历该 `slob` 上的可用空间:

```
for (prev = NULL, cur = sp->freelist; ; prev = cur, cur = slob_next(cur)) {
```

对应每个可用的空间,首先查看其是否有足够的空间,
不过,如果有地址对齐的要求,需要把用于对齐的空间
也考虑进去

```
slobidx_t avail = slob_units(cur);

if (align) {
	aligned = (slob_t *)ALIGN((unsigned long)cur, align);
	delta = aligned - cur;
}
if (avail >= units + delta) { /* room enough? */
```

如果有足够的空间,那么我们便可以从当前节点中去分配了.
不过,如果之前有用于对齐的空间,需要将其剔除.

```
if (delta) { /* need to fragment head to align? */
	next = slob_next(cur);
	set_slob(aligned, avail - delta, next);
	set_slob(cur, delta, aligned);
	prev = cur;
	cur = aligned;
	avail = slob_units(cur);
}
```

下面就是比较该空间的大小和所需的大小,
我们先来看该空间的大小正好等于所需的大小的情形,
这时,只需将该空间从 `freelist` 中拿出即可:

```
next = slob_next(cur);
if (avail == units) { /* exact fit? unlink. */
	if (prev)
		set_slob(prev, slob_units(prev), next);
	else
		sp->freelist = next;
```

反之,说明该空间大于需要的空间,
这时只需将我们所需的空间取出,
并将剩下的空间放入到 `freelist`:

```
} else { /* fragment */
	if (prev)
		set_slob(prev, slob_units(prev), cur + units);
	else
		sp->freelist = cur + units;
	set_slob(cur + units, avail - units, next);
}
```

最终调整该 `slob` 可用的空间大小.
如果该 `slob` 没有空闲的空间了,
将其从对应的链表中删除.

```
sp->units -= units;
if (!sp->units)
	clear_slob_page_free(sp);
```

```
static inline void clear_slob_page_free(struct page *sp)
{
	list_del(&sp->list);
	__ClearPageSlobFree(sp);
}
```

至此,申请流程结束.

## Freeing a object back to cache

下面我们来看一个 `object` 是如何释放的,
这里首先还是判断释放的 `object` 的大小,
如果其大于一个物理页,直接返回给 `zone` `page` `allocator`,
否则,返回给 `slob`:

```
if (size < PAGE_SIZE)
	slob_free(b, size);
else
	slob_free_pages(b, get_order(size));
```

这里,我们只关心后一点.
首先找到该 `object` 对应的物理页,
如果该物理页除了该 `object` 都被释放了,
那么,我们直接将该物理页释放给 `zone` `page` `allocator`:

```
sp = virt_to_page(block);
units = SLOB_UNITS(size);

spin_lock_irqsave(&slob_lock, flags);

if (sp->units + units == SLOB_UNITS(PAGE_SIZE)) {
	/* Go directly to page allocator. Do not pass slob allocator */
	if (slob_page_free(sp))
		clear_slob_page_free(sp);
	spin_unlock_irqrestore(&slob_lock, flags);
	__ClearPageSlab(sp);
	page_mapcount_reset(sp);
	slob_free_pages(b, 0);
	return;
}
```

如果该物理页上的 `objects` 之前从没被释放过,
说明此次释放的 `object` 为第一个,
那么该 `slob` 的状态即将变为部分空闲.
那么初始化该 `slob`,并根据 `object` 的大小,
放入到对应的全局链表中.

```
if (!slob_page_free(sp)) {
	/* This slob page is about to become partially free. Easy! */
	sp->units = units;
	sp->freelist = b;
	set_slob(b, units,
		(void *)((unsigned long)(b +
				SLOB_UNITS(PAGE_SIZE)) & PAGE_MASK));
	if (size < SLOB_BREAK1)
		slob_list = &free_slob_small;
	else if (size < SLOB_BREAK2)
		slob_list = &free_slob_medium;
	else
		slob_list = &free_slob_large;
	set_slob_page_free(sp, slob_list);
	goto out;
}
```

如果该物理页的状态已经为部分空闲了,
那么我们就需要在其 `freelist` 上找到合适的位置,
将该 `object` 放进去,
这里需要注意的是 `freelist` 上的空闲 `objects`
按照地址大小顺序排列的.
我们首先来考虑,释放的 `object` 的地址小于 `freelist`
上的第一个 `object`,
这时我们需要将释放的 `object` 插入到 `freelist` 的头部,
而且,如果该节点和其之后的节点可以合并,首先将他们合并.

```
if (b < (slob_t *)sp->freelist) {
	if (b + units == sp->freelist) {
		units += slob_units(sp->freelist);
		sp->freelist = slob_next(sp->freelist);
	}
	set_slob(b, units, sp->freelist);
	sp->freelist = b;
```

如果该 `object` 的地址大于 `freelist` 上的第一个节点,
那么我们首先在 `freelist` 上找到需要插入的位置的前一个
和后一个节点.

```
prev = sp->freelist;
next = slob_next(prev);
while (b > next) {
	prev = next;
	next = slob_next(prev);
}
```

接着将其插入,这里在插入之前同样考虑合并的可能性:

```
if (!slob_last(prev) && b + units == next) {
	units += slob_units(next);
	set_slob(b, units, slob_next(next));
} else
	set_slob(b, units, next);

if (prev + slob_units(prev) == b) {
	units = slob_units(b) + slob_units(prev);
	set_slob(prev, units, slob_next(b));
} else
	set_slob(prev, slob_units(prev), b);
```

至此,释放流程结束.

FIN.
