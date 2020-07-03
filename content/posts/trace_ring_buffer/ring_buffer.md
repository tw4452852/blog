---
categories:
- learning
date: 2015-04-26T16:49:00
tags:
- linux
- kernel
- trace
title: Ring buffer design
---

## 简介

本文主要分析下 `kernel` 中的 `ring` `buffer` 的设计与实现.
它主要被应用到 `kernel` 中的 `trace` 框架.

## 应用模型

`ring` `buffer` 有两种可用的使用模式:

- 生产消费模式: 当生产者没有足够空间写入新的内容,那么该内容将被丢弃,直到消费者读取了 `buffer` 中的一些内容,所以在 `buffer` 满的情况下,会丢弃最新的内容.
- 复写模式: 该模式与上面的区别是,当 `buffer` 满时,最旧的内容将被最新的内容替代.

## 读写同步

对于 `buffer` 读写,有如下规则:

- 同一时间不允许多个写者对同一块 `buffer` 进行操作,但是不同 `cpu` 上写者对该 `cpu` 上自己的 `buffer` 进行操作(每个 `cpu` 有自己独立的 `buffer`), 但是一个写者可以被另一个写者打断,当其完成操作之后,再返回到之前被打断的写者. 这里和中断的模型很像,整个顺序和 `stack` 很像:

```
writer1 start
 <preempted> writer2 start
     <preempted> writer3 start
                 writer3 finishes
             writer2 finishes
writer1 finishes
```

- 同一时间不允许多个读者同时操作,读者之前也不能相互打断,但是读者可以和不同 `cpu` 上的写者同时操作同一块 `buffer` ,读者可以被写者打断.

## 数据模型

`ring` `buffer` 的实现,主要是由一个双向链表组成的物理页,其中几个主要的指针:

- `reader` `page:` 指向读者正在操作的 `page` ,该页不在 `ring` `buffer` 的链表当中, 但是可以被换入到链表中.

- `head` `page:` 指向链表中的下一个被读者换出的 `page` .

- `tail` `page:` 指向链表中下一次写操作的 `page` .

- `commit` `page:` 指向链表中最后一些写操作完成(因为写操作可能会被打断)所在的 `page` .

```
  +------+
  |reader|
  |page  |
  +------+
                             tail page
  head page         commit page  |
      |                 |        |
      v                 v        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
	         RING BUFFER
```

下面具体分析读、写操作.

## 写操作

由于写操作可能被打断,这里采取的策略是每次写操作之前都会事先预留写的空间,
当写操作完成时,才将该空间 `commit` ,如果之间有被其他的写者打断,
那么只有最早的写者才会最终修改 `commit` 指针, 使其内容对读者可见.

```
Write reserve:

   Buffer page
  +---------+
  |written  |
  +---------+  <--- given back to writer (current commit)
  |reserved |
  +---------+ <--- tail pointer
  | empty   |
  +---------+

Write commit:

   Buffer page
  +---------+
  |written  |
  +---------+
  |written  |
  +---------+  <--- next position for write (current commit)
  | empty   |
  +---------+


If a write happens after the first reserve:

   Buffer page
  +---------+
  |written  |
  +---------+  <-- current commit
  |reserved |
  +---------+  <--- given back to second writer
  |reserved |
  +---------+ <--- tail pointer

After second writer commits:


   Buffer page
  +---------+
  |written  |
  +---------+  <--(last full commit)
  |reserved |
  +---------+
  |pending  |
  |commit   |
  +---------+ <--- tail pointer

When the first writer commits:

   Buffer page
  +---------+
  |written  |
  +---------+
  |written  |
  +---------+
  |written  |
  +---------+  <--(last full commit and tail pointer)
```

所以在代码中,一次写操作的主要步骤是:
`rb_start_commit->write_content->rb_end_commit`

可以看出这里的 `rb_start_commit` 和 `rb_end_commit` 保证操作的一致型.

```
static void rb_start_commit(struct ring_buffer_per_cpu *cpu_buffer)
{
	local_inc(&cpu_buffer->committing);
	local_inc(&cpu_buffer->commits);
}
```

这里有两个标记量:

- `committing:` 表示正在进行的写者的数量.
- `commits:` 需要 `commit` 的数量.

在 `rb_end_commit` 中,我们首先获得当前写者的数量,如果只剩下了一个
(也就是当前进程自己),那么我们需要将所写的内容进行 `commit` ,
使其对读者可见.

```
commits = local_read(&cpu_buffer->commits);
/* synchronize with interrupts */
barrier();
if (local_read(&cpu_buffer->committing) == 1)
	rb_set_commit_to_write(cpu_buffer);
```

之后我们将 `committing` 还原.

```
local_dec(&cpu_buffer->committing);

/* synchronize with interrupts */
barrier();
```

不过,这里还是有一个问题,试想这样一种情况:
如果在恢复 `committing` 之前被另一个写者打断,
而它又同样在执行 `rb_end_commit` ,
这时它会发现 `committing>1` ,那么它并不会 `commit`
它所写的内容,而此时我们只是 `commit` 当前我们自己的内容,
那么这样我们就丢失一次内容的 `commit` ,所以这里为了防止
这种情况的发生,我们重新读取 `commits` 和之前的比较,
如果发生变化,说明这种情况发生了,我们会再进行一次 `commit`

```
if (unlikely(local_read(&cpu_buffer->commits) != commits) &&
    !local_read(&cpu_buffer->committing)) {
	local_inc(&cpu_buffer->committing);
	goto again;
}
```

下面我们如果来进行写操作.

```
tail_page = cpu_buffer->tail_page;
write = local_add_return(length, &tail_page->write);

/* set write to only the index of the write */
write &= RB_WRITE_MASK;
tail = write - length;
```

这里,我们首先获得写操作所在的页( `tail_page`),
以及本次写操作在页内的起始偏移( `tail` )和结束偏移( `write`),
注意,这里我们已经保留了这段空间,因为下一次的页内偏移已经增加
( `tail_page->write` )

如果所写内容仍然在当前页上,那么只需要将 `tail` 所指向的空间返回即可.

但是如果所写的区域跨越了当前页,那么我们就需要将内容写在新的页上,
同时将 `tail_page` 向后移一个,也就是这里 `rb_move_tail` 所做的事.

这里就是这个设计里面比较 `tricky` 的地方了.

首先获得 `tail_page` 下一个页的指针:

```
next_page = tail_page;

rb_inc_page(cpu_buffer, &next_page);
```

下面,有个极端情况需要考虑:
正常情况下, `tail_page` 在 `commit_page` 之后或者相同,
但是如果不断有中断来打断最开始的那个写者,那么 `tail_page`
就不断的后移,直到它的下一个 `page` 就是 `commit_page` ,
在这种情况下,我们不能再将 `tail_page` 后移,
因为它将把之前所写的内容冲掉.

```
if (unlikely(next_page == commit_page)) {
	local_inc(&cpu_buffer->commit_overrun);
	goto out_reset;
}
```

下面,我们需要判断之后的那个 `page` 是否是 `head_page` ,
为什么需要进行这个判断呢,因为读者和写者是可以同时运行的,
而读者会将之前的 `page` 换入到链表中,而换入的目标正是 `head_page` ！

所以为了防止读者换入操作和当前的写操作冲突,
我们需要提前将 `head_page` 后移,整个流程如下:

```
            tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
                        ^
                        |
                    head page


            tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
                                 ^
                                 |
                             head page


                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
                                 ^
                                 |
                             head page
```

下面我们还需要考虑 `commit_page` 是否和 `reader_page` 在同一个 `page` 上,
因为 `reader_page` 并不在链表中,所以之前为了防止数据被冲掉而做的检查,
这里同样也需要:

```
          reader page    commit page
              |              |
              v              |
             +---+           |
             |   |<----------+
             |   |<------------ tail page
             |   |------+
             +---+      |
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-H->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

a lot of interrupts here and move tail page ahead ...

          reader page    commit page
              |              |
              v              |
             +---+           |
             |   |<----------+
             |   |
             |   |------+
             +---+      |
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-H->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
               ^
               |
           tail page
```

```
if (unlikely((cpu_buffer->commit_page !=
	      cpu_buffer->tail_page) &&
	     (cpu_buffer->commit_page ==
	      cpu_buffer->reader_page))) {
	local_inc(&cpu_buffer->commit_overrun);
	goto out_reset;
}
```

下面就剩下一种情况:
`reader_page` 和 `commit_page` 不在同一个 `page` 上,
同时 `tail_page` 的下一个 `page` 就是 `head_page` .

这里,如果当前的模式为生产消费模型,那么需要直接丢弃当前的内容:

```
if (!(buffer->flags & RB_FL_OVERWRITE)) {
	local_inc(&cpu_buffer->dropped_events);
	goto out_reset;
}
```

如果不是,那么我们就需要丢弃最旧的数据.
这里,我们先分析现在可能的竞争:

- 读者可能将 `head_page` 换出
- 别的写者可能打断当前的进程,往后写入更多的数据,从而将 `tail_page` 后移.

所以,这里问题的关键是将 `head_page` 后移和可能的 `page` 换入的原子性.
这也是将 `ring` `buffer` `lockless` 的关键.

这里,我们采用的方法是将两个标记存入到指向 `head_page` 的指针中,
也就是 `head_page->prev->next`
这两个标记如下:

- `HEADER:` 该指针所指向的是 `head_page`
- `UPDATE:` 该指针所指向的 `page` 正在被写者更新,它之前或者即将成为 `head_page`

这里为了能够将这两个标记存入到指针中,所以所有的 `page`
数据结构(这里为了和物理页区分)都必须至少是4字节对齐(最后两位用于存放标记).

### case 1

当 `tail_page` 之后即为 `head_page` ,那么它首先将 `tail_page->next` 的标记从
`HEAD` 变为 `UPDATE`

```
            tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-H->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

            tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

当读者尝试换入 `page` 时,它会检查指向 `head_page` 的指针是否含有 `HEADER`
标记,如果没有,那么它会不断尝试,这里就防止了读者的竞争.

接着,我们将 `head_page` 推进一个:

```
           tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

然后将之前的标记从 `UPDATE` 清除:

```
           tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

最终,将 `tail_page` 指向新的 `page` :

```
                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

### case 2

这是一般的情形,让我们考虑一个稍微复杂一点的情况,当第一个写者将标记从 `HEADER`
变为 `UPDATE` 之后:

```
first writer:
            tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

这时,另一个写者打断了当前写者,这时它发现之前的标记已经是 `UPDATE` ,
那么它会接着将之后的指针标记为 `HEADER`

```
second writer:
           tail page
               |
               v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

但是它并不会将之前的标记从 `UPDATE` 去除,因为那并不是它设置的,
这里它只是将 `tail_page` 向前移动一个:

```
second writer:
                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

当该写者返回,会到最初的的写者,那么它只需要将之前设置的 `UPDATE` 标记去除即可:

```
first writer:
                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

### case 3

其实,还有个更复杂的情况,试想在第二个写者还没有更新 `tail_page` 之前:

```
second writer:
                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-H->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

这时,另一个写者打断进来,它又将 `tail_page` 和 `head_page` 各向前推进了一个,
具体步骤如下:

```
third writer:

                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-U->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-U->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

                    tail page
                        |
                        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

                             tail page
                                 |
                                 v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

接着返回到第二个写者,这是它发现更新 `tail_page` 失败,
因为它已经被第三个写者更新了.

最后,回到第一个写者,
这时它并不知道 `tail_page` 已经移动了,它仍旧设置它认为的 `HEADER` 标记:


```
first writer:
                             tail page
                                 |
                                 v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+

                             tail page
                                 |
                                 v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |-H->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

但是从这里可以看出,这样的处理不够好,
第一个写者还需要判断 `tail_page` 是否已经移动,
如果 `tail_page` 既不是 `A` 也不是 `B` ,
那么它需要将之前设置的它所认为的指向假的 `head_page` 的 `HEADER` 标记清除.

```
first writer:

               A        B    tail page
               |        |        |
               v        v        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

最终将它设置的 `UPDATE` 标记清除:

```
first writer:
               A        B    tail page
               |        |        |
               v        v        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |-U->|   |--->|   |-H->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```

好了,这就是所有可能的情形,搞清楚这三种可能性,再来看代码就比较容易了,
详见 `rb_handle_head_page` 函数,这里就不再赘述了.

## 读操作

下面来看下读操作的代码,首先是大部分的情况,当前 `reader_page` 上还有足够的数据,
那么我们直接返回当前的 `reader_page` 即可:

```
if (cpu_buffer->reader_page->read < rb_page_size(reader))
	goto out;
```

反之,我们需要换出一个新的 `page` ,不过在这之前,我们需要考虑如果 `commit_page`
还在当前的 `reader_page` 上,那么我们就不能进行换出操作,只能返回 `NULL`

```
reader = NULL;
if (cpu_buffer->commit_page == cpu_buffer->reader_page)
	goto out;
```

下面,我们就可以尝试进行换出了,首先是获得 `head_page`

```
reader = rb_set_head_page(cpu_buffer);
if (!reader)
	goto out;
```

接着将当前的 `reader_page->next` 指向 `head_page->next`,
`reader_page->prev` 指向 `head_page->prev`

```
cpu_buffer->reader_page->list.next = rb_list_head(reader->list.next);
cpu_buffer->reader_page->list.prev = reader->list.prev;
```

当然,我们还需要设置 `HEADER` 标记:

```
rb_set_list_to_head(cpu_buffer, &cpu_buffer->reader_page->list);
```

整体结构如下图:

```
  +------+
  |reader|          RING BUFFER
  |page  |-------H-----------+
  +------+                   v
    |             +---+    +---+    +---+
    |             |   |--->|   |--->|   |
    |             |   |<---|   |<---|   |<-+
    |             +---+    +---+    +---+  |
    |              ^ |               ^ |   |
    |              | +---------------+ |   |
    |              +-----H-------------+   |
    +--------------------------------------+
```

接着需要将之前指向 `head_page` 的指针指向当前的 `reader_page`
如果失败了,说明其他的写者正在移动 `head_page` ,那么我们将重新进行尝试,
直至成功.

```
ret = rb_head_page_replace(reader, cpu_buffer->reader_page);
```

```
  +------+
  |reader|          RING BUFFER
  |page  |-------H-----------+
  +------+                   v
    |  ^          +---+   +---+   +---+
    |  |          |   |-->|   |-->|   |
    |  |          |   |<--|   |<--|   |<-+
    |  |          +---+   +---+   +---+  |
    |  |             |             ^ |   |
    |  |             +-------------+ |   |
    |  +-----------------------------+   |
    +------------------------------------+
```

成功之后,下一步就需要将 `head_page->next->prev` 指向当前的 `reader_page`
这样当前的 `reader_page` 就被挂入链表了.

```
rb_list_head(reader->list.next)->prev = &cpu_buffer->reader_page->list;
```

```
  +------+
  |reader|          RING BUFFER
  |page  |-------H-----------+
  +------+ <---------------+ v
    |  ^          +---+   +---+   +---+
    |  |          |   |-->|   |-->|   |
    |  |          |   |   |   |<--|   |<-+
    |  |          +---+   +---+   +---+  |
    |  |             |             ^ |   |
    |  |             +-------------+ |   |
    |  +-----------------------------+   |
    +------------------------------------+
```

最终更新 `head_page` 以及 `reader_page`

```
rb_inc_page(cpu_buffer, &cpu_buffer->head_page);

/* Finally update the reader page to the new head */
cpu_buffer->reader_page = reader;
```

```
  +------+
  |buffer|          RING BUFFER
  |page  |-------H-----------+  <--- New head page
  +------+ <---------------+ v
    |  ^          +---+   +---+   +---+
    |  |          |   |   |   |-->|   |
    |  |  New     |   |   |   |<--|   |<-+
    |  | Reader   +---+   +---+   +---+  |
    |  |  page ----^                 |   |
    |  |                             |   |
    |  +-----------------------------+   |
    +------------------------------------+
```

之后在新的 `reader_page` 上读取数据.

FIN.
