+++
title = "Temporary kernel mapping"
date = "2014-03-02T11:57:00"
tags = [ "linux", "kernel" ]
categories = ["learning"]
+++

## 简介

本篇紧接着[上一篇]({{< ref ulk_pkmap >}}),
继续讲解kernel中对高端内存访问的第二种技术:
Temporary mapping.

从上一讲我们可以看出,permanent mapping是有可能阻塞的,
等待其他线程释放一些页表资源,显然,在一些场景下,
这种等待是不能接受,比如在中断回调函数中.

这样,temporary mapping就被引入进来,它是非阻塞的,
当然,它也有它的不足:页表的生命周期是暂时.

## 窗口

和permanent mapping类似,我们在kernel地址空间的顶端
预留了一些窗口(实质是页表项),专门用于temporary mapping,
当然这些窗口的个数是有限.

其中,每个CPU只有一定数量(KM_TYPE_NR)的窗口供其使用:

```
#ifdef __WITH_KM_FENCE
# define KM_TYPE_NR 41
#else
# define KM_TYPE_NR 20
#endif
```

这里的__WITH_KM_FENCE是为了做debug用.

那么,我们如何通过窗口得到它所对应的虚拟地址,
或者通过虚拟地址得到对应的窗口呢.
这里有两个宏分别与之对应:

```
#define __fix_to_virt(x)	(FIXADDR_TOP - ((x) << PAGE_SHIFT))
#define __virt_to_fix(x)	((FIXADDR_TOP - ((x)&PAGE_MASK)) >> PAGE_SHIFT)
```

这里的FIXADDR_TOP可以先认为就是可寻址的最大地址
(32bit: `2**32`, 64bit: `2**64`).

说到这里,就不能不说下fix mapping了,
顾名思义,他建立的映射的,
为什么会有这样的映射呢,因为有些地址是有特殊用途的,
而且也是固定的,那么我们就必须将这些地址保留下来,
不能让它用于其他的映射,
在kernel中,这些地址都有一个枚举变量与之对应:

```
enum fixed_addresses {
#ifdef CONFIG_X86_32
	FIX_HOLE,
	FIX_VDSO,
#else
	VSYSCALL_LAST_PAGE,
	VSYSCALL_FIRST_PAGE = VSYSCALL_LAST_PAGE
			    + ((VSYSCALL_END-VSYSCALL_START) >> PAGE_SHIFT) - 1,
	VVAR_PAGE,
	VSYSCALL_HPET,
#endif
#ifdef CONFIG_PARAVIRT_CLOCK
	PVCLOCK_FIXMAP_BEGIN,
	PVCLOCK_FIXMAP_END = PVCLOCK_FIXMAP_BEGIN+PVCLOCK_VSYSCALL_NR_PAGES-1,
#endif
	FIX_DBGP_BASE,
	FIX_EARLYCON_MEM_BASE,
#ifdef CONFIG_PROVIDE_OHCI1394_DMA_INIT
	FIX_OHCI1394_BASE,
#endif
#ifdef CONFIG_X86_LOCAL_APIC
	FIX_APIC_BASE,	/* local (CPU) APIC) -- required for SMP or not */
#endif
#ifdef CONFIG_X86_IO_APIC
	FIX_IO_APIC_BASE_0,
	FIX_IO_APIC_BASE_END = FIX_IO_APIC_BASE_0 + MAX_IO_APICS - 1,
#endif
#ifdef CONFIG_X86_VISWS_APIC
	FIX_CO_CPU,	/* Cobalt timer */
	FIX_CO_APIC,	/* Cobalt APIC Redirection Table */
	FIX_LI_PCIA,	/* Lithium PCI Bridge A */
	FIX_LI_PCIB,	/* Lithium PCI Bridge B */
#endif
	FIX_RO_IDT,	/* Virtual mapping for read-only IDT */
#ifdef CONFIG_X86_32
	FIX_KMAP_BEGIN,	/* reserved pte's for temporary kernel mappings */
	FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1,
#ifdef CONFIG_PCI_MMCONFIG
	FIX_PCIE_MCFG,
#endif
#endif
#ifdef CONFIG_PARAVIRT
	FIX_PARAVIRT_BOOTMAP,
#endif
	FIX_TEXT_POKE1,	/* reserve 2 pages for text_poke() */
	FIX_TEXT_POKE0, /* first page is last, because allocation is backward */
#ifdef	CONFIG_X86_INTEL_MID
	FIX_LNW_VRTC,
#endif
	__end_of_permanent_fixed_addresses,

	/*
	 * 256 temporary boot-time mappings, used by early_ioremap(),
	 * before ioremap() is functional.
	 *
	 * If necessary we round it up to the next 256 pages boundary so
	 * that we can have a single pgd entry and a single pte table:
	 */
#define NR_FIX_BTMAPS		64
#define FIX_BTMAPS_SLOTS	4
#define TOTAL_FIX_BTMAPS	(NR_FIX_BTMAPS * FIX_BTMAPS_SLOTS)
	FIX_BTMAP_END =
	 (__end_of_permanent_fixed_addresses ^
	  (__end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS - 1)) &
	 -PTRS_PER_PTE
	 ? __end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS -
	   (__end_of_permanent_fixed_addresses & (TOTAL_FIX_BTMAPS - 1))
	 : __end_of_permanent_fixed_addresses,
	FIX_BTMAP_BEGIN = FIX_BTMAP_END + TOTAL_FIX_BTMAPS - 1,
#ifdef CONFIG_X86_32
	FIX_WP_TEST,
#endif
#ifdef CONFIG_INTEL_TXT
	FIX_TBOOT_BASE,
#endif
	__end_of_fixed_addresses
};
```

当然,其中就有为temporary mapping预留的地址空间:

```
FIX_KMAP_BEGIN,	/* reserved pte's for temporary kernel mappings */
FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1,
```

所以只要我们有了这个枚举变量,通过上述的两个宏,
就可以得到其对应的虚拟地址,反之亦然.

## 申请和释放

申请和释放对应的API分别为kmap_atomic和kunmap_atomic.
对于申请和释放,最主要的工作就是找到一个空闲的页表项,
和permanent mapping不同,这里不能等待.
在具体的实现中,是通过一个原子操作得到这个index的:

申请时:

```
type = kmap_atomic_idx_push();
```

释放时:

```
kmap_atomic_idx_pop();
```

这里每个cpu有自己的一个全局变量,用于标识当前使用的index,

```
DECLARE_PER_CPU(int, __kmap_atomic_idx);
```

当申请时,只是将该计数原子+1,如果其超过了最大的可用个数,
那么kernel认为这是一个bug.

```
static inline int kmap_atomic_idx_push(void)
{
	int idx = __this_cpu_inc_return(__kmap_atomic_idx) - 1;

#ifdef CONFIG_DEBUG_HIGHMEM
	WARN_ON_ONCE(in_irq() && !irqs_disabled());
	BUG_ON(idx > KM_TYPE_NR);
#endif
	return idx;
}
```

同理,释放时该值原子-1:

```
static inline void kmap_atomic_idx_pop(void)
{
#ifdef CONFIG_DEBUG_HIGHMEM
	int idx = __this_cpu_dec_return(__kmap_atomic_idx);

	BUG_ON(idx < 0);
#else
	__this_cpu_dec(__kmap_atomic_idx);
#endif
}
```

有了这个index,下面就是得到其在全局的枚举变量中的位置:

```
idx = type + KM_TYPE_NR*smp_processor_id();
```

以及其对应的虚拟地址:

```
vaddr = __fix_to_virt(FIX_KMAP_BEGIN + idx);
```

最终更新对应的页表项:

```
BUG_ON(!pte_none(*(kmap_pte-idx)));
set_pte(kmap_pte-idx, mk_pte(page, prot));
```

这里有两点需要注意:

- 如果该页表项不为空,说明有人正在使用,那么kernel认为这是个bug.
- 全局变量kmap_pte对应用于temporary mapping的最后一个页表项.

释放的过程和申请的过程同理,首先是得到当前使用的页表项在全局枚举变量中的位置:

```
type = kmap_atomic_idx();
idx = type + KM_TYPE_NR * smp_processor_id();
```

接着是清空对应的页表项:

```
kpte_clear_flush(kmap_pte-idx, vaddr);
```

最终释放用于表示当前CPU使用的index的计数:

```
kmap_atomic_idx_pop();
```

FIN.
