+++
title = "Linux kernel boot process - part 3"
date = "2017-03-01T16:33:00"
tags = [ "linux", "kernel" ]
+++

## background

[上一篇]({{< ref "linux_kernel_boot_process_2" >}})
我们已经将compressed kernel解压到了指定位置 , 
下面我们就接着来看看uncompressed kernel的启动过程 . 

## checking

首先是初始化stack , 并查看cpu是否支持64位 : 

```
/* Set up the stack for verify_cpu(), similar to initial_stack below */
leaq	(__end_init_task - SIZEOF_PTREGS)(%rip), %rsp

/* Sanitize CPU configuration */
call verify_cpu
```

其实 , 在之前我们已经验证过cpu是否支持64位 , 
这里再次检查是因为有可能我们是通过64位的bootloader跳转到这里 , 
从这段代码的上面的comment , 我们也可以得到验证 : 

```
* We come here either directly from a 64bit bootloader, or from
* arch/x86/boot/compressed/head_64.S.
```

接着计算当前位置和编译时的位置之间的差值 : 

```
/*
 * Compute the delta between the address I am compiled to run at and the
 * address I am actually running at.
 */
leaq	_text(%rip), %rbp
subq	$_text - __START_KERNEL_map, %rbp
...
#define __START_KERNEL_map 0xffffffff80000000
```

而编译时的位置可以从 `arch/x86/kernel/vmlinux.lds` 中得到 : 

```
SECTIONS
{
 . = (0xffffffff80000000 + ALIGN(0x1000000, 0x200000));
 phys_startup_64 = ABSOLUTE(startup_64 - 0xffffffff80000000);
 /* Text and read-only data */
 .text : AT(ADDR(.text) - 0xffffffff80000000) {
  _text = .;
```

所以 , 这里的差值为0 (0x1000000 - (0xffffffff80000000 + 0x1000000 -
0xffffffff80000000)).

接着 , 检查地址的合法性 , 包括是否2M对其 , 是否超过64位cpu支持的最大地址 : 

```
/* Is the address not 2M aligned? */
testl	$~PMD_PAGE_MASK, %ebp
jnz	bad_address

/*
 * Is the address too large?
 */
leaq	_text(%rip), %rax
shrq	$MAX_PHYSMEM_BITS, %rax
jnz	bad_address
```

## early page mapping

从 `arch/x86/kernel/vmlinux.lds` 可以看出编译时的线性地址从 `0xffffffff80000000`
开始 , 而真实的物理地址却是从0开始 , 
所以 ，这里需要建立页表将其映射起来 . 

下面我们就来看看这部分页表的结构 , 
由于我们采用的4级页表的结构 , 首先需要计算出该线性地址在pgdt中的位置 : 

(0xffffffff80000000 >> 39) & 511 = 511

其次是在pudt中的位置 : 

(0xffffffff80000000 >> 30) & 511 = 510

接着是在pmdt中的位置 : 

(0xffffffff80000000 >> 21) & 511 = 0

清楚了这些位置 ，下面就是对应的页表初始化 : 

```
NEXT_PAGE(early_level4_pgt)
	.fill	511,8,0
	.quad	level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE
NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)
NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
	/* 8MB reserved for vsyscalls + a 2MB hole = 4 + 1 entries */
	.fill	5,8,0
NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
...
#define KERNEL_IMAGE_SIZE	(512 * 1024 * 1024)
#define PMDS(START, PERM, COUNT)			\
	i = 0 ;						\
	.rept (COUNT) ;					\
	.quad	(START) + (i << PMD_SHIFT) + (PERM) ;	\
	i = i + 1 ;					\
	.endr
```

可以看出这里将虚拟地址空间 ( 0xffffffff80000000 ~ 0xffffffff80000000+512M )
映射到物理地址空间 ( 0 ~ 512M ) .

由于该页表目录结构是在编译时初始化的 ,
所以这里还需要将其地址调整为实际运行的地址 , 
这里 , 只需要加上之前计算出的差值即可 : 

```
/*
 * Fixup the physical addresses in the page table
 */
addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

除了建立上述的地址映射 , kernel还建立一个对等映射 , 
即线性地址对应和其相同的物理地址 :

```
/*
 * Set up the identity mapping for the switchover.  These
 * entries should *NOT* have the global bit set!  This also
 * creates a bunch of nonsense entries but that is fine --
 * it avoids problems around wraparound.
 */
leaq	_text(%rip), %rdi
leaq	early_level4_pgt(%rip), %rbx
movq	%rdi, %rax
shrq	$PGDIR_SHIFT, %rax
leaq	(PAGE_SIZE + _KERNPG_TABLE)(%rbx), %rdx
movq	%rdx, 0(%rbx,%rax,8)
movq	%rdx, 8(%rbx,%rax,8)
addq	$PAGE_SIZE, %rdx
movq	%rdi, %rax
shrq	$PUD_SHIFT, %rax
andl	$(PTRS_PER_PUD-1), %eax
movq	%rdx, PAGE_SIZE(%rbx,%rax,8)
incl	%eax
andl	$(PTRS_PER_PUD-1), %eax
movq	%rdx, PAGE_SIZE(%rbx,%rax,8)
addq	$PAGE_SIZE * 2, %rbx
movq	%rdi, %rax
shrq	$PMD_SHIFT, %rdi
addq	$(__PAGE_KERNEL_LARGE_EXEC & ~_PAGE_GLOBAL), %rax
leaq	(_end - 1)(%rip), %rcx
shrq	$PMD_SHIFT, %rcx
subq	%rdi, %rcx
incl	%ecx
1:
andq	$(PTRS_PER_PMD - 1), %rdi
movq	%rax, (%rbx,%rdi,8)
incq	%rdi
addq	$PMD_SIZE, %rax
decl	%ecx
jnz	1b
```

可以看出这里将虚拟地址空间 (_text(0x1000000) ~ _end) 映射到对等的物理地址空间 . 

之后 , 根据kernel是否被relocation , 对页表中的地址做调整 : 

```
leaq	level2_kernel_pgt(%rip), %rdi
leaq	PAGE_SIZE(%rdi), %r8
/* See if it is a valid page table entry */
1:	testb	$_PAGE_PRESENT, 0(%rdi)
jz	2f
addq	%rbp, 0(%rdi)
/* Go to the next page */
2:	addq	$8, %rdi
cmp	%r8, %rdi
jne	1b
```

最终 , 物理地址和虚拟地址的对应关系如下 : 

```
        physical address           virtual address
          +-------+                 +-------+
          |       |                 |       |
          |       |     +-----------+-------+ 0xffffffff80000000+512M
     512M +-------+<----+           |       |
          |       |                 |       |
          |       |     +-----------+-------+ 0xffffffff80000000
          |       |     |           |       |
          |       |     |           |       |
          |       |     |           |       |
          |       |     |           |       |
     _end +-------+<----+-----------+-------+ _end
          | kernel|     |           |       |
          | image |     |           |       |
0x1000000 +-------+<----+-----------+-------+ 0x1000000
          |       |     |           |       |
          |       |     |           |       |
          |       |     |           |       |
          |       |     |           |       |
      0x0 +-------+<----+           +-------+
```

之后打开pae和pge , 并加载新的页表 : 

```
	movq	$(early_level4_pgt - __START_KERNEL_map), %rax
	jmp 1f
...
1:
	/* Enable PAE mode and PGE */
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	/* Setup early boot stage 4 level pagetables. */
	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

这样 , 我们就能调转到编译时指定的虚拟地址 : 

```
	/* Ensure I am executing from virtual addresses */
	movq	$1f, %rax
	jmp	*%rax
1:
```

之后 , kernel会重新设置cr0 , 初始化stack和各个段寄存器 . 
最终， 跳入到c代码 `x86_64_start_kernel` : 

```
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
	...
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
```

FIN.
