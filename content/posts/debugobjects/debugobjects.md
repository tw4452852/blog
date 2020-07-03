---
categories:
- learning
date: 2017-03-15T09:54:00
tags:
- linux
- kernel
title: Debug objects framework
---

## background

Debug objects ( 以下简称do ) 是linux kernel对object生命周期追踪和操作验证的通用框架 .
它常被用于检查下面这些常见bug : 

1. 使用未初始化的object . 

2. 初始化正在使用的object . 

3. 使用已被销毁的object . 

下面结合代码来看看具体如何使用以及其内部的实现原理 . 


## api

do对外提供一些列的api , 用于外部模块告知object生命周期的变化 : 

1. `debug_object_init:` 用于告知object已被初始化 , 当该object正在被使用 ,
或者是之前已被销毁 , 那么do会给出warning , 同时 , 如果外部模块提供修正接口 , 
do会尝试修正 . 当然如果该object是第一次初始化 , 那么do会将其状态置位已初始化 . 
这里需要注意的是 , 该接口用于那些分配在heap上的object , 如果do发现该object
在调用者的stack上 , 同样会给出warning . 

2. `debug_object_init_on_stack:` 和 `debug_object_init` 类似 , 只不过该object
被初始化在stack上 .

3. `debug_object_activate:` 告知object正在被使用 , 如果该object已经被使用 ,
或者之前已被销毁， 那么do会给出warning , 并尝试修正 . 

4. `debug_object_deactivate:` 告知object停止使用 , 如果该object已经被停用 , 
或者已被销毁 , 那么do会给出warning.

5. `debug_object_destroy:` 告知object被销毁 , 如果该object正在被使用 ,
或者已被销毁 , 那么do会给出warning , 并尝试修正该错误 .

6. `debug_object_free:` 告知object被释放 , 如果该object正在被使用 , 
那么do会给出warning , 并尝试修正该错误 . 同时该接口还会将object在do中
的记录移除 . 

7. `debug_object_assert_init:` 用于断言object已被初始化 , 如果发现该object
还未被初始化 , 那么do会给出warning , 并尝试初始化该object . 

8. `debug_check_no_obj_freed:` 该接口用于检查一段已经释放的地址空间是否还存在
未被释放的object . 如果发现还有正在使用的object , 那么会将其释放并给出warning ,
若存在未被使用的object , 那么会将其记录从do中删除 . 

## implement

do内部对object的抽象结构为 `struct` `debug_obj:`

```
/**
 * struct debug_obj - representaion of an tracked object
 * @node:	hlist node to link the object into the tracker list
 * @state:	tracked object state
 * @astate:	current active state
 * @object:	pointer to the real object
 * @descr:	pointer to an object type specific debug description structure
 */
struct debug_obj {
    struct hlist_node	node;
    enum debug_obj_state	state;
    unsigned int		astate;
    void			*object;
    struct debug_obj_descr	*descr;
};
```

这些 `debug_obj` 被存放在全局的hash table中 : 

```
struct debug_bucket {
    struct hlist_head	list;
    raw_spinlock_t		lock;
};

#define ODEBUG_HASH_BITS	14
#define ODEBUG_HASH_SIZE	(1 << ODEBUG_HASH_BITS)
static struct debug_bucket	obj_hash[ODEBUG_HASH_SIZE];
```

这里为每个bucket设置了一个lock是为了减少lock的contention . 

而hash的key的则为实际的object的地址/page_size :

```
#define ODEBUG_CHUNK_SHIFT	PAGE_SHIFT
static struct debug_bucket *get_bucket(unsigned long addr)
{
    unsigned long hash;

    hash = hash_long((addr >> ODEBUG_CHUNK_SHIFT), ODEBUG_HASH_BITS);
    return &obj_hash[hash];
}
```

所以同一个页内的object都在同一个bucket中 . 

而每个 `debug_obj` 都是从 `obj_pool` 中分配 , 
该pool最多容纳1024个 `debug_obj` , 最小需要保留256个 `debug_obj` : 

```
#define ODEBUG_POOL_SIZE	1024
#define ODEBUG_POOL_MIN_LEVEL	256
static DEFINE_RAW_SPINLOCK(pool_lock);
static HLIST_HEAD(obj_pool);
```

当pool中的 `debug_obj` 过多 , 会将多余的释放 `kmem_cache`  :

```
static void free_obj_work(struct work_struct *work);
static DECLARE_WORK(debug_obj_work, free_obj_work);
static void free_obj_work(struct work_struct *work)
{
	struct debug_obj *obj;
	unsigned long flags;

	raw_spin_lock_irqsave(&pool_lock, flags);
	while (obj_pool_free > ODEBUG_POOL_SIZE) {
		obj = hlist_entry(obj_pool.first, typeof(*obj), node);
		hlist_del(&obj->node);
		obj_pool_free--;
		/*
		 * We release pool_lock across kmem_cache_free() to
		 * avoid contention on pool_lock.
		 */
		raw_spin_unlock_irqrestore(&pool_lock, flags);
		kmem_cache_free(obj_cache, obj);
		raw_spin_lock_irqsave(&pool_lock, flags);
	}
	raw_spin_unlock_irqrestore(&pool_lock, flags);
}

static void free_object(struct debug_obj *obj)
{
    ...
	if (obj_pool_free > ODEBUG_POOL_SIZE && obj_cache)
		sched = 1;
	...
	if (sched)
		schedule_work(&debug_obj_work);
}
```

如果pool中空闲的 `debug_obj` 过少 , 那么会从 `kmem_cache` 中申请 : 

```
static void fill_pool(void)
{
    ...
    while (obj_pool_free < ODEBUG_POOL_MIN_LEVEL) {

        new = kmem_cache_zalloc(obj_cache, gfp);
        if (!new)
            return;

        raw_spin_lock_irqsave(&pool_lock, flags);
        hlist_add_head(&new->node, &obj_pool);
        obj_pool_free++;
        raw_spin_unlock_irqrestore(&pool_lock, flags);
    }
}

static void
__debug_object_init(void *addr, struct debug_obj_descr *descr, int onstack)
{
    ...
	fill_pool();
    ...
	obj = lookup_object(addr, db);
	if (!obj) {
		obj = alloc_object(addr, db, descr);
    ...
}
```

最后 , 关于do系统的初始化是分两部分来完成的 ,
最开始 `obj_pool` 是静态分配的 , 当kernel `kmem_caches` 使能之后 , 
所有的 `debug_obj` 都是动态分配的 , 之前静态的 `debug_obj` 会被释放 , 
取而代之的是动态分配一个用于替换原来的 . 

```
asmlinkage __visible void __init start_kernel(void)
{
    ...
	debug_objects_early_init();
    ...
	debug_objects_mem_init();
    ...
}

static struct debug_obj		obj_static_pool[ODEBUG_POOL_SIZE] __initdata;

void __init debug_objects_early_init(void)
{
	int i;

	for (i = 0; i < ODEBUG_HASH_SIZE; i++)
		raw_spin_lock_init(&obj_hash[i].lock);

	for (i = 0; i < ODEBUG_POOL_SIZE; i++)
		hlist_add_head(&obj_static_pool[i].node, &obj_pool);
}

void __init debug_objects_mem_init(void)
{
	obj_cache = kmem_cache_create("debug_objects_cache",
				      sizeof (struct debug_obj), 0,
				      SLAB_DEBUG_OBJECTS, NULL);

	if (!obj_cache || debug_objects_replace_static_objects()) {
		debug_objects_enabled = 0;
		if (obj_cache)
			kmem_cache_destroy(obj_cache);
		pr_warn("out of memory.\n");
	} else
		...
}
```

下面来看看 `debug_objects_replace_static_objects` 的实现 . 

首先从 `kmem_cache` 中动态分配空闲的 `debug_obj` : 

```
HLIST_HEAD(objects);

for (i = 0; i < ODEBUG_POOL_SIZE; i++) {
    obj = kmem_cache_zalloc(obj_cache, GFP_KERNEL);
    if (!obj)
        goto free;
    hlist_add_head(&obj->node, &objects);
}
```

接着 , 将之前静态分配的 `debug_obj` 从 `obj_pool` 中删除 , 并将当前动态分配的
`debug_obj` 放进去 : 

```
/* Remove the statically allocated objects from the pool */
hlist_for_each_entry_safe(obj, tmp, &obj_pool, node)
    hlist_del(&obj->node);
/* Move the allocated objects to the pool */
hlist_move_list(&objects, &obj_pool);
```

最后 , 如果之前已经有了object被do记录 , 需要将这些object替换成我们动态申请的
`debug_obj` , 这里 , 通过遍历全局的hash table便可以得到当前有记录的object : 

```
/* Replace the active object references */
struct debug_bucket *db = obj_hash;
for (i = 0; i < ODEBUG_HASH_SIZE; i++, db++) {
    hlist_move_list(&db->list, &objects);

    hlist_for_each_entry(obj, &objects, node) {
        new = hlist_entry(obj_pool.first, typeof(*obj), node);
        hlist_del(&new->node);
        /* copy object data */
        *new = *obj;
        hlist_add_head(&new->node, &db->list);
        cnt++;
    }
}
```

## use case

最后 , 我们通过一个例子来看看如何使用do . 

这里模拟的场景是 : 释放一个正在使用object . 

```
typedef struct test_obj {
    int i;
} test_obj_s;

struct test_obj *s = kmalloc(sizeof(*s), GFP_KERNEL);

if (!s) {
    pr_err("oom !\n");
    return -1;
}
debug_object_init(s, &test_obj_debug_descr);
debug_object_activate(s, &test_obj_debug_descr);

/* free a active object */
kfree(s);
```

使用do时 , 需要提供一个用于描述object的descriptor ,
其中 , 我们可以提供相关的修正函数 , 当do尝试修正某个错误时 , 
对应的callback会被调用 , 下面是该descriptor的定义 : 

```
/**
 * struct debug_obj_descr - object type specific debug description structure
 *
 * @name:		name of the object typee
 * @debug_hint:		function returning address, which have associated
 *			kernel symbol, to allow identify the object
 * @is_static_object:	return true if the obj is static, otherwise return false
 * @fixup_init:		fixup function, which is called when the init check
 *			fails. All fixup functions must return true if fixup
 *			was successful, otherwise return false
 * @fixup_activate:	fixup function, which is called when the activate check
 *			fails
 * @fixup_destroy:	fixup function, which is called when the destroy check
 *			fails
 * @fixup_free:		fixup function, which is called when the free check
 *			fails
 * @fixup_assert_init:  fixup function, which is called when the assert_init
 *			check fails
 */
struct debug_obj_descr {
    const char		*name;
    void *(*debug_hint)(void *addr);
    bool (*is_static_object)(void *addr);
    bool (*fixup_init)(void *addr, enum debug_obj_state state);
    bool (*fixup_activate)(void *addr, enum debug_obj_state state);
    bool (*fixup_destroy)(void *addr, enum debug_obj_state state);
    bool (*fixup_free)(void *addr, enum debug_obj_state state);
    bool (*fixup_assert_init)(void *addr, enum debug_obj_state state);
};
```

这里 , 为了简单起见 , 我们只设置该object的名字以及当释放出错时的修正函数 : 

```
static bool
test_obj_fixup_free(void *addr, enum debug_obj_state state)
{
    pr_err("%s: state[%#x], addr[%p]\n", __func__, state, addr);
    debug_object_deactivate(addr, &test_obj_debug_descr);
    return false;
}

static struct debug_obj_descr test_obj_debug_descr = {
    .name = "test obj",
    .fixup_free = test_obj_fixup_free,
};
```

在释放修正函数中 , 我们强制设置object的状态为未使用 . 

这样 , 当程序运行时 , kernel会出现如下warning : 

```
WARNING: CPU: 0 PID: 30 at lib/debugobjects.c:263 debug_print_object+0x85/0xa0
ODEBUG: free active (active state 0) object type: test obj hint: 0xffff8a3f45739328
Modules linked in: tm(+)
CPU: 0 PID: 30 Comm: modprobe Not tainted 4.10.0-rc6-g79c9089-dirty #29
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.8.1-0-g4adadbd-20150316_085822-nilsson.home.kraxel.org 04/01/2014
Call Trace:
 dump_stack+0x19/0x1d
 __warn+0xc5/0xe0
 warn_slowpath_fmt+0x4a/0x50
 ? put_dec+0x1a/0x80
 debug_print_object+0x85/0xa0
 debug_check_no_obj_freed+0x190/0x1c0
 ? init_tw+0x57/0x6a [tm]
 kfree+0xcf/0x120
 ? 0xffffffffc007f000
 init_tw+0x57/0x6a [tm]
 do_one_initcall+0x3e/0x170
 ? __vunmap+0xa6/0xf0
 ? kfree+0xcf/0x120
 do_init_module+0x55/0x1c4
 load_module+0x1da2/0x2190
 ? __symbol_get+0x60/0x60
 ? kernel_read+0x3b/0x50
 SyS_finit_module+0xa0/0xb0
 entry_SYSCALL_64_fastpath+0x13/0x93
RIP: 0033:0x496559
RSP: 002b:00007ffc7a53f768 EFLAGS: 00000246 ORIG_RAX: 0000000000000139
RAX: ffffffffffffffda RBX: 0000000000859e00 RCX: 0000000000496559
RDX: 0000000000000000 RSI: 0000000000632f11 RDI: 0000000000000008
RBP: 00007f2f436ea010 R08: 000000000084ab98 R09: 7fffffffffffffff
R10: 0000000000000030 R11: 0000000000000246 R12: 00007ffc7a53f760
R13: 0000000000871fc2 R14: 00007f2f43705010 R15: 00007ffc7a541f9d
---[ end trace 34366a9b991a12f7 ]---
test_obj_fixup_free: state[0x3], addr[ffff8a3f45739328]
```

可以看出 , do检测出我们正在free一个正被使用的object(status = 3 =
ODEBUG_STATE_ACTIVE) , 同时将call stack和相关
的context也dump出来 , 这样我们就可以很方便的定位出问题出现的位置 .

FIN.
