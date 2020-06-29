+++
title = "DRM Memory Ranger Allocator"
date = "2014-12-14T16:06:00"
tags = ["linux", "kernel", "drm", "gfx"]
+++

## 基本概念

`DRM` 提供了一个简单的基于地址空间的分配器,以下简称 `drm_mm` .
该分配器提供了最基本的空间的申请和释放的功能.

在开始之前,我们必须先清楚一个概念:空洞.
在 `drm_mm` 看来为被占用的地址空间都称为空洞.
而随着不断的新的 `buffer` 的申请,
与之大小对应的地址空间被占用,
所以空洞会越来越小,
反之,如果有一些 `buffer` 被释放了,
那么其占用的地址空间也就被释放了,
而这些空间就重新变成了空洞.
随着反复的申请和释放,空洞就会遍布在整个地址空间,
这里如果就会牵涉到空间的融合,也就是说,相邻的两个空
会合并成一个大的空洞.
当然一个大的空洞,也有可能因为在其中分配一个 `buffer`
而被分割成两个小的空洞.

下面我们来看下它的基本数据结构.
首先,最主要是的数据结构为 `drm_mm` ,它用于抽象一个分配器:

```
struct drm_mm {
	struct list_head hole_stack; // 所有含有空洞的节点的链表
	struct drm_mm_node head_node; // 所有的已分配的节点的链表, 按照起始地址升序排序
	struct list_head unused_nodes; // drm_mm_node结构体的cache链表
	int num_unused; // cache链表中的个数
	spinlock_t unused_lock; // 用于保护cache链表
	unsigned int scan_check_range : 1; // 用户是否指定地址的范围
	unsigned scan_alignment; // 检查的对齐要求
	unsigned long scan_color; // 检查的color设置
	unsigned long scan_size; // 希望的空洞的大小
	unsigned long scan_hit_start; // 标记找到的空洞的起始地址
	unsigned long scan_hit_end; // 标记找到的空洞的尾地址
	unsigned scanned_blocks; // 记录经过检查的节点的个数
	unsigned long scan_start; // 用户指定的范围的开始地址
	unsigned long scan_end; // 用户指定的范围的结束地址
	struct drm_mm_node *prev_scanned_node; // 所有经过检查的节点的链表

	void (*color_adjust)(struct drm_mm_node *node, unsigned long color,
			     unsigned long *start, unsigned long *end); // 对符合的条件的节点进行可选的地址范围的调整
};
```

这里有个检查的概念,后续会有详细介绍,这里暂且跳过.

下面我们来看下另一个数据结构,也就是 `drm_mm_node`
,用于抽象一个已经分配的地址范围:

```
struct drm_mm_node {
	struct list_head node_list; // 在drm_mm的全局链表中的位置
	struct list_head hole_stack; // 若该节点后面存在空洞，在表示在drm_mm的hole_stack的位置
	unsigned hole_follows : 1; // 该节点后面是否存在空洞
	unsigned scanned_block : 1; // 检查器是否检查过该节点
	unsigned scanned_prev_free : 1; // 暂时没用
	unsigned scanned_next_free : 1; // 暂时没用
	unsigned scanned_preceeds_hole : 1; // 保存在检查之前该节点的前一个节点是否有空洞，用于状态恢复
	unsigned allocated : 1; // 该节点是否被分配使用
	unsigned long color; // 用于地址调整的颜色值
	unsigned long start; // 占用的地址范围的开始
	unsigned long size; // 占用的地址范围的结束
	struct drm_mm *mm; // 指向所属的分配器
};
```

## 分配

分配一个地址范围主要分为两步:

- 查找一个满足条件的空洞
- 从空洞中剔除分配的地址范围

下面我们先来看如何查找一个满足条件的空洞,
这里的条件主要有需要的地址空间的大小,对齐要求,
以及可能存在的对空间的起始以及结束地址的要求.
具体的实现分别在 `drm_mm_search_free_in_range_generic`
和 `drm_mm_search_free_generic` 中,
两者的逻辑一样,只不过一个是对地址范围的有要求,
而另一个没有.

```
best = NULL;
best_size = ~0UL;

drm_mm_for_each_hole(entry, mm, adj_start, adj_end) {
	if (adj_start < start)
		adj_start = start;
	if (adj_end > end)
		adj_end = end;

	if (mm->color_adjust) {
		mm->color_adjust(entry, color, &adj_start, &adj_end);
		if (adj_end <= adj_start)
			continue;
	}

	if (!check_free_hole(adj_start, adj_end, size, alignment))
		continue;

	if (!(flags & DRM_MM_SEARCH_BEST))
		return entry;

	if (entry->size < best_size) {
		best = entry;
		best_size = entry->size;
	}
}

return best;
```

可以看出这里遍历所有的空洞,从中尝试寻找.
如果指定 `DRM_MM_SEARCH_BEST` 标记,
那么就找出满足条件的最小的空洞.

下面便是将分配的地址空间从空洞中剔除,
在代码中也就是将分配的节点插入到空洞节点中.

这里,首先确定的分配的地址空间的范围,
这里包括可选的驱动实现的地址范围微调以及
为了满足对齐要求的调整:

```
if (mm->color_adjust)
	mm->color_adjust(hole_node, color, &adj_start, &adj_end);

if (alignment) {
	unsigned tmp = adj_start % alignment;
	if (tmp)
		adj_start += alignment - tmp;
}
```

接下来,如果分配的空间的起始地址正好与空洞的其实地址相同,
换句话说,也就是正好在空洞的开头,那么原先的节点后面就不
存在空洞了,需要将其从 `hole_stack` 中去除:

```
if (adj_start == hole_start) {
	hole_node->hole_follows = 0;
	list_del(&hole_node->hole_stack);
}
```

下面就是初始化新分配的节点的属性,包括起始地址,
大小等等,并将其挂入空洞节点的后面.

```
node->start = adj_start;
node->size = size;
node->mm = mm;
node->color = color;
node->allocated = 1;

INIT_LIST_HEAD(&node->hole_stack);
list_add(&node->node_list, &hole_node->node_list);
```

最后,需要查看新分配的节点后面是否还存在空洞,
如果存在的话,还需要将该节点挂入到全局的 `hole_stack` 中:

```
node->hole_follows = 0;
if (__drm_mm_hole_node_start(node) < hole_end) {
	list_add(&node->hole_stack, &mm->hole_stack);
	node->hole_follows = 1;
}
```

## 释放

释放时,如果该节点在 `hole_stack` 中,
首先将其去除:

```
if (node->hole_follows) {
	BUG_ON(__drm_mm_hole_node_start(node) ==
	       __drm_mm_hole_node_end(node));
	list_del(&node->hole_stack);
} else
	BUG_ON(__drm_mm_hole_node_start(node) !=
	       __drm_mm_hole_node_end(node));
```

接着,如果前节点原先不存在空洞,那么现在由于该节点的释放而
存在了空洞,需要将其添加到 `hole_stack` 中,反之,将前节点
移动到 `hole_stack` 的开头,以便使其有更大的机会被分配.

```
if (!prev_node->hole_follows) {
	prev_node->hole_follows = 1;
	list_add(&prev_node->hole_stack, &mm->hole_stack);
} else
	list_move(&prev_node->hole_stack, &mm->hole_stack);
```

最终,将该节点标记为未分配,并将其从全局节点链表中删除.

```
list_del(&node->node_list);
node->allocated = 0;
```

## 其他

这里还有两个接口,一个是用于保留一些地址空间,
因为一些 `firmware` 会保留一些地址空间:
`drm_mm_reserve_node` ,其中的逻辑和在一个空洞中插入一个
分配的节点一样,这里不再赘述.

另一个为 `drm_mm_reserve_node` ,主要用于节点的替换.

## LRU Scan/Eviction

由于很多 `GPU` 要求使用的内存对象其地址是连续的,
因此当我们因为需要一个比较大的连续地址空间,
或者对空间有严格要求时而造成很多没必要的零碎的小空间被回收.

为了解决这个问题, `drm_mm` 提出了检查器(或者叫收集器)的概念,
当检查到一个满足条件的空洞之后,只会回收那些在该空洞中的节点,
这样就不会造成不必要的回收.

下面我来看看这个检查器是如何工作的.
首先在开始检查之前,我们需要对检查器的状态进行初始化,
包括需要收集连续空间的大小、范围、对齐要求等等.

```
mm->scan_color = color;
mm->scan_alignment = alignment;
mm->scan_size = size;
mm->scanned_blocks = 0;
mm->scan_hit_start = 0;
mm->scan_hit_end = 0;
mm->scan_start = start;
mm->scan_end = end;
mm->scan_check_range = 1;
mm->prev_scanned_node = NULL;
```

初始化之后,下面我们将可以被回收的节点依次放入到检查器中.
首先将该节点标记为被检查的状态,同时更新检查器检查的节点的个数:

```
mm->scanned_blocks++;

BUG_ON(node->scanned_block);
node->scanned_block = 1;
```

接着,我们当前节点从其前节点后面删除,并将其挂入到全局的扫描链表中,
并进行相应的状态保存(因为之后,我们需要进行状态的恢复):

```
node->scanned_preceeds_hole = prev_node->hole_follows;
prev_node->hole_follows = 1;
list_del(&node->node_list);
node->node_list.prev = &prev_node->node_list;
node->node_list.next = &mm->prev_scanned_node->node_list;
mm->prev_scanned_node = node;
```

下面,我们可以检查前节点后面的空洞大小,
注意,之前我们已经将当前节点从节点后面去掉,
所以这时的空洞中,其实已经包含了当前节点.

```
adj_start = hole_start = drm_mm_hole_node_start(prev_node);
adj_end = hole_end = drm_mm_hole_node_end(prev_node);
```

然后进行相应的空间范围的调整,从而最终确定空洞的范围:

```
if (mm->scan_check_range) {
	if (adj_start < mm->scan_start)
		adj_start = mm->scan_start;
	if (adj_end > mm->scan_end)
		adj_end = mm->scan_end;
}

if (mm->color_adjust)
	mm->color_adjust(prev_node, mm->scan_color,
			 &adj_start, &adj_end);
```

最后,检查该空洞是否满足分配的要求,
如果满足则记录下当前的空洞范围,并返回1,
反之则范围0.

```
if (check_free_hole(adj_start, adj_end,
		    mm->scan_size, mm->scan_alignment)) {
	mm->scan_hit_start = hole_start;
	mm->scan_hit_end = hole_end;
	return 1;
}

return 0;
```

经过一系列的检查,如果我们找到了满足条件的空洞,
或者没有找到,我们还需要将那些节点的状态重新恢复回去,
(毕竟它们已经不在全局的节点链表中了),从而最终恢复
`drm_mm` 的状态.

这里主要是将之前的保存的状态还原,并将该节点重新
挂入到全局链表的合适位置:

```
mm->scanned_blocks--;

BUG_ON(!node->scanned_block);
node->scanned_block = 0;

prev_node = list_entry(node->node_list.prev, struct drm_mm_node,
		       node_list);

prev_node->hole_follows = node->scanned_preceeds_hole;
list_add(&node->node_list, &prev_node->node_list);
```

最终,如果该节点落在符合的空洞中则返回1,
反之则返回0.

```
return (drm_mm_hole_node_end(node) > mm->scan_hit_start &&
 node->start < mm->scan_hit_end);
```

FIN.
