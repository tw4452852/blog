---
categories:
- Understand Linux Kernel
date: 2017-03-13T14:20:00
tags:
- linux
- kernel
title: Kernel address space layout randomization
---

## background

KASLR(Kernel address space layout randomization) 用于将kernel地址随机化 , 
这里 , 既包含物理地址也包含对应的虚拟地址 . 

## decompress kernel

kaslr 在解压kernel到内存之前会随机选择大小合适的物理地址 ,
下面我们就来看看这部分是如何实现的 . 

首先 , 我们需要将一些已知的物理地址空间过滤掉 ,
因为这些空间已经存放了内容 . 

这里 , 每个区域都是由如下的数据结构表示 : 

```
struct mem_vector {
    unsigned long start;
    unsigned long size;
};
```

而已知的需要过滤的区域由如下几个 , 它们都保存在一个静态的全局变量中 : 

```
enum mem_avoid_index {
    MEM_AVOID_ZO_RANGE = 0,
    MEM_AVOID_INITRD,
    MEM_AVOID_CMDLINE,
    MEM_AVOID_BOOTPARAMS,
    MEM_AVOID_MAX,
};

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

1. compressed kernel + decompress code所占用的空间 :
这里主要是用于解压kernel时所需的空间 . 

```
mem_avoid[MEM_AVOID_ZO_RANGE].start = input;
mem_avoid[MEM_AVOID_ZO_RANGE].size = (output + init_size) - input;
```

2. initrd

```
initrd_start  = (u64)boot_params->ext_ramdisk_image << 32;
initrd_start |= boot_params->hdr.ramdisk_image;
initrd_size  = (u64)boot_params->ext_ramdisk_size << 32;
initrd_size |= boot_params->hdr.ramdisk_size;
mem_avoid[MEM_AVOID_INITRD].start = initrd_start;
mem_avoid[MEM_AVOID_INITRD].size = initrd_size;
```

3. kernel command line

```
cmd_line  = (u64)boot_params->ext_cmd_line_ptr << 32;
cmd_line |= boot_params->hdr.cmd_line_ptr;
/* Calculate size of cmd_line. */
ptr = (char *)(unsigned long)cmd_line;
for (cmd_line_size = 0; ptr[cmd_line_size++]; )
    ;
mem_avoid[MEM_AVOID_CMDLINE].start = cmd_line;
mem_avoid[MEM_AVOID_CMDLINE].size = cmd_line_size;
```

4. 启动相关信息 ( zero page )

```
mem_avoid[MEM_AVOID_BOOTPARAMS].start = (unsigned long)boot_params;
mem_avoid[MEM_AVOID_BOOTPARAMS].size = sizeof(*boot_params);
```

下面就是找出所有符合大小的可用的物理地址空间 ,
这里主要是遍历bios提供物理内存表 ( e820 ) . 

```
/* Verify potential e820 positions, appending to slots list. */
for (i = 0; i < boot_params->e820_entries; i++) {
    process_e820_entry(&boot_params->e820_map[i], minimum,
               image_size);
    ...
}
```

对于每个e820表项 , 首先判断其是否为可用内存 : 

```
/* Skip non-RAM entries. */
if (entry->type != E820_RAM)
    return;
```

接着 , 如果是32位kernel， 最多只能将kernel放置在512M以下的空间
所以需要判断该表项的其实地址是否大于512M : 

```
#define KERNEL_IMAGE_SIZE	(512 * 1024 * 1024)

/* On 32-bit, ignore entries entirely above our maximum. */
if (IS_ENABLED(CONFIG_X86_32) && entry->addr >= KERNEL_IMAGE_SIZE)
    return;
```

其次 , 判断该区域小于指定的起始地址 : 

```
#define KERNEL_IMAGE_SIZE	(512 * 1024 * 1024)

/* On 32-bit, ignore entries entirely above our maximum. */
if (IS_ENABLED(CONFIG_X86_32) && entry->addr >= KERNEL_IMAGE_SIZE)
    return;
```

通过了这些检查 , 那么该区域可能满足条件 . 
下面还需要对其起始地址做对其 , 并重新计算该区域的大小 : 

```
/* Potentially raise address to meet alignment needs. */
region.start = ALIGN(region.start, CONFIG_PHYSICAL_ALIGN);

/* Did we raise the address above this e820 region? */
if (region.start > entry->addr + entry->size)
    return;

/* Reduce size by any delta from the original address. */
region.size -= region.start - start_orig;

/* On 32-bit, reduce region size to fit within max size. */
if (IS_ENABLED(CONFIG_X86_32) &&
    region.start + region.size > KERNEL_IMAGE_SIZE)
    region.size = KERNEL_IMAGE_SIZE - region.start;

/* Return if region can't contain decompressed kernel */
if (region.size < image_size)
    return;
```

如果该区域足够容纳我们所需的大小 , 下一步便是检查
该区域是否和我们之前定义的需要过滤的空间有重叠 ,
如果没有重叠 , 那么该区域便成为候选之一 : 

```
/* If nothing overlaps, store the region and return. */
if (!mem_avoid_overlap(&region, &overlap)) {
    store_slot_info(&region, image_size);
    return;
}
```

反之 , 如果有重叠 , 但是在重叠区域之前由足够的空间 , 
这里同样将该区域记录下来 : 

```
/* Store beginning of region if holds at least image_size. */
if (overlap.start > region.start + image_size) {
    struct mem_vector beginning;

    beginning.start = region.start;
    beginning.size = overlap.start - region.start;
    store_slot_info(&beginning, image_size);
}
```

否则 , 从该区域中去除重叠的部分 , 调整区域的起始地址和大小 ,
然后进行下一次的比较 : 

```
/* Return if overlap extends to or past end of region. */
if (overlap.start + overlap.size >= region.start + region.size)
    return;

/* Clip off the overlapping region and start over. */
region.size -= overlap.start - region.start + overlap.size;
region.start = overlap.start + overlap.size;
```

最终 , 所有的候选区域都被保存在全局 : 

```
#define MAX_SLOT_AREA 100
static struct slot_area slot_areas[MAX_SLOT_AREA];

static void store_slot_info(struct mem_vector *region, unsigned long image_size)
{
    ...
	slot_area.addr = region->start;
	slot_area.num = (region->size - image_size) /
			CONFIG_PHYSICAL_ALIGN + 1;
    ...
}
```

这里 , 每个区域处理会保存该区域的起始地址 , 
同时还保存了该区域可以容纳的镜像的个数 . 

最后 , 我们就需要从这些候选区域中随机选择一个 : 

```
static unsigned long slots_fetch_random(void)
{
    ...
	slot = kaslr_get_random_long("Physical") % slot_max;

	for (i = 0; i < slot_area_index; i++) {
		if (slot >= slot_areas[i].num) {
			slot -= slot_areas[i].num;
			continue;
		}
		return slot_areas[i].addr + slot * CONFIG_PHYSICAL_ALIGN;
	}

	if (i == slot_area_index)
		debug_putstr("slots_fetch_random() failed!?\n");
	return 0;
}
```

## identity map

从 [之前分析x86_64启动过程]({{< ref linux_kernel_boot_process_2 >}})
可知，在进入保护模式之后我们只映射了0到4G的地址空间,
那么, 我们这里随机选择的地址可能还没有映射 ,
所以 , 这里还需要建立对应的页表 , 是decompress code可以访问这些内存地址 . 

这里 , 会建立1对1的映射 ，而建立页表的过程会依次遍历pgd , pud , pmd
如果对应的页表项不存在 , 会重新创建一个 , 并插入到对应的表项中 : 

```
void add_identity_map(unsigned long start, unsigned long size)
{
	/* Build the mapping. */
	kernel_ident_mapping_init(&mapping_info, (pgd_t *)level4p,
				  start, end);
}
int kernel_ident_mapping_init(struct x86_mapping_info *info, pgd_t *pgd_page,
			      unsigned long pstart, unsigned long pend)
{
	for (; addr < end; addr = next) {
		pgd_t *pgd = pgd_page + pgd_index(addr);
        ...
		if (pgd_present(*pgd)) {
			pud = pud_offset(pgd, 0);
			result = ident_pud_init(info, pud, addr, next);
			...
			continue;
		}
		pud = (pud_t *)info->alloc_pgt_page(info->context);
		...
		result = ident_pud_init(info, pud, addr, next);
		...
		set_pgd(pgd, __pgd(__pa(pud) | _KERNPG_TABLE));
	}
    ...
}
static int ident_pud_init(struct x86_mapping_info *info, pud_t *pud_page,
			  unsigned long addr, unsigned long end)
{
	unsigned long next;

	for (; addr < end; addr = next) {
		pud_t *pud = pud_page + pud_index(addr);
		if (pud_present(*pud)) {
			pmd = pmd_offset(pud, 0);
			ident_pmd_init(info, pmd, addr, next);
			continue;
		}
		pmd = (pmd_t *)info->alloc_pgt_page(info->context);
		ident_pmd_init(info, pmd, addr, next);
		set_pud(pud, __pud(__pa(pmd) | _KERNPG_TABLE));
	}
}
static void ident_pmd_init(struct x86_mapping_info *info, pmd_t *pmd_page,
			   unsigned long addr, unsigned long end)
{
	addr &= PMD_MASK;
	for (; addr < end; addr += PMD_SIZE) {
		pmd_t *pmd = pmd_page + pmd_index(addr);

		if (pmd_present(*pmd))
			continue;

		set_pmd(pmd, __pmd((addr - info->offset) | info->pmd_flag));
	}
}
```

那么 , 这些页表项是存放在哪的呢 ? 我们可以从页表分配函数中看出端倪 : 

```
void initialize_identity_maps(void)
{
	/* Init mapping_info with run-time function/buffer pointers. */
	mapping_info.alloc_pgt_page = alloc_pgt_page;
	mapping_info.context = &pgt_data;
    ...
}
static void *alloc_pgt_page(void *context)
{
    ...
	entry = pages->pgt_buf + pages->pgt_buf_offset;
	pages->pgt_buf_offset += PAGE_SIZE;

	return entry;
}
```

可见 , 这些页表项在内存中是连续存放的 ,
下面是相关成员的初始化的值 : 

```
void initialize_identity_maps(void)
{
    ...
	pgt_data.pgt_buf_offset = 0;
	/*
	 * If we came here via startup_32(), cr3 will be _pgtable already
	 * and we must append to the existing area instead of entirely
	 * overwriting it.
	 */
	level4p = read_cr3();
	if (level4p == (unsigned long)_pgtable) {
		debug_putstr("booted via startup_32()\n");
		pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
		pgt_data.pgt_buf_size = BOOT_PGT_SIZE - BOOT_INIT_PGT_SIZE;
		memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
	} else {
		debug_putstr("booted via startup_64()\n");
		pgt_data.pgt_buf = _pgtable;
		pgt_data.pgt_buf_size = BOOT_PGT_SIZE;
		memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
		level4p = (unsigned long)alloc_pgt_page(&pgt_data);
	}
}
```

这里 , 会判断我们是来自于64位的bootloader还是32位的 , 
如果是32位的话 , 我们已经建立的0-4G的空间对应的页表项, 所以这里需要保留这些页表项 , 
如果是64位的话 , 我们需要重新建立所有的页表项 . 

```
// vmlinux.lds.S
       .pgtable : {
		_pgtable = . ;
		*(.pgtable)
		_epgtable = . ;
	}

// arch/x86/kernel/head_64.S
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill BOOT_PGT_SIZE, 1, 0

# define BOOT_INIT_PGT_SIZE	(6*4096)
#  ifdef CONFIG_X86_VERBOSE_BOOTUP
#   define BOOT_PGT_SIZE	(19*4096)
#  else /* !CONFIG_X86_VERBOSE_BOOTUP */
#   define BOOT_PGT_SIZE	(17*4096)
#  endif
```

## kernel virtual mapping randomization

除了对kernel image的物理地址的随机化 , kaslr同时还可以对kernel virtual
address的随机化 . 

这里主要对三块区域进行随机化 : 

1. kernel virtual address

2. vmalloc

3. vmemmap

```
static __initdata struct kaslr_memory_region {
    unsigned long *base;
    unsigned long size_tb;
} kaslr_regions[] = {
    { &page_offset_base, 64/* Maximum */ },
    { &vmalloc_base, VMALLOC_SIZE_TB },
    { &vmemmap_base, 1 },
};
```

每个区域都会随机出一个偏移量加到其默认的初始值上 : 

```
void __init kernel_randomize_memory(void)
{
    ...
	for (i = 0; i < ARRAY_SIZE(kaslr_regions); i++) {
		unsigned long entropy;

		/*
		 * Select a random virtual address using the extra entropy
		 * available.
		 */
		entropy = remain_entropy / (ARRAY_SIZE(kaslr_regions) - i);
		prandom_bytes_state(&rand_state, &rand, sizeof(rand));
		entropy = (rand % (entropy + 1)) & PUD_MASK;
		vaddr += entropy;
		*kaslr_regions[i].base = vaddr;

		/*
		 * Jump the region and add a minimum padding based on
		 * randomization alignment.
		 */
		vaddr += get_padding(&kaslr_regions[i]);
		vaddr = round_up(vaddr + 1, PUD_SIZE);
		remain_entropy -= entropy;
	}
}
```

FIN.
