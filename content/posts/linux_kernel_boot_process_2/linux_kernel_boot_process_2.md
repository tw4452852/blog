+++
title = "Linux kernel boot process - part 2"
date = "2017-02-27T15:01:00"
tags = [ "linux", "kernel" ]
+++

## background

这里继续[上一篇]({{< ref "linux_kernel_boot_process_1" >}})
分析kernel从protect mode到long mode的过程.

在开始之前 , 有必要了解下kernel被bootloader加载到内存的位置 : 

```
+---------------+ <------------------------------------------------------------+
|               |                                                              |
| protected-mode|                                                              |
|   kernel      |                                                              |
+---------------+0x100000 <--------------------+                               |
|               |                              |                               |
|               |                              |                               |
|               |                              |                               |
+---------------+X+setup_code_size <----------+|                               |
| kernel setup  |                             ||                               |
|               |                             ||                               |
+---------------+X  <------------+            ||                               |
| Boot loader   |                |            ||                               |
+---------------+0x1000          |            ||                               |
| reserved      |                +------------++-------------------------------+
+---------------+0x0800          |header+setup|       compressed kernel        |
| used by MBR   |                +------------+--------------------------------+
+---------------+0x0600
| BIOS use only |
+---------------+0x0000
```

这里的 `X` , 不同的bootloader可能不同 , 当前在我本地为 0x10000 .

下面我们就从protect mode的第一条指令开始分析 . 

## prepare to jump to long mode

由于我们测试的是x86_64的kernel , 所以 , 首个指令位于
`arch/x86/boot/compressed/header_64.S` 中 , 
首先是clear direction 标记 : 

```
__HEAD
.code32
ENTRY(startup_32)
/*
 * 32bit entry is 0 and it is ABI so immutable!
 * If we come here directly from a bootloader,
 * kernel(text+data+bss+brk) ramdisk, zero_page, command line
 * all need to be under the 4G limit.
 */
cld
```

接着 , 检查bootloader是否要求保存段寄存器的值 , 
如果不需要 , 将ds , es , ss 段寄存器初始化位ds段描述符.

```
/*
 * Test KEEP_SEGMENTS flag to see if the bootloader is asking
 * us to not reload segments
 */
testb $KEEP_SEGMENTS, BP_loadflags(%esi)
jnz 1f

cli
movl	$(__BOOT_DS), %eax
movl	%eax, %ds
movl	%eax, %es
movl	%eax, %ss
```

下面需要计算出当前运行的地址和编译时的地址的差值 , 
有了这个差值 , 后续我们就能根据编译时的地址得到当前运行时真实的物理地址 , 
反之亦然 . 

这里采用的方法是通过 call 指令得到后一条指令的地址 : 

```
leal	(BP_scratch+4)(%esi), %esp
call	1f
1:	popl	%ebp
subl	$1b, %ebp
```

这里通过使用kernel header中的4字节的通用字段充当临时的stack , 
当调用call指令时 , cpu会将call指令的后一条指令的物理地址push到stack上 . 
之后我们就可能同stack上取道其物理地址 , 最后和其编译时地址相减便得到其差值 . 
该差值最终保存在ebp寄存器中 : 

```
+----------+ <-esp      +--------------------------+
|  empty   |            |runtime address of label 1|
|  stack   | -------->  +--------------------------+ <-esp
|          | after call |                          |
+----------+            +--------------------------+
```

得到了该差值 , 紧接着便是调整stack寄存器 : 

```
/* setup a stack and make sure cpu supports long mode. */
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
	...
/*
 * Stack and heap for uncompression
 */
	.bss
	.balign 4
...
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

接着检查cpu是否支持64位 : 

```
call	verify_cpu
testl	%eax, %eax
jnz	no_longmode
```

由于kernel支持relocation , 这里还需要计算出relocate的目标地址 :

```
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
```

默认情况下relocate的基地址为 `LOAD_PHYSICAL_ADDR` ( 0x1000000 ) .
该地址用于存放解压之后的kernel . 

同时 , 这里还需要将解压代码和压缩的kernel拷贝到relocate目标区域的尾部 .

```
/* Target address to relocate to for decompression */
movl	BP_init_size(%esi), %eax
subl	$_end, %eax
addl	%eax, %ebx
```

可以看出 , 这里通过查看kernel header中的init_size字段 ,
获得relocate区域大小 , 再减去 `_end` 即可 . 

```

         |<----------------  init_size ----------------------------------->|
         |                                                                 |
   relocation address                 |<----------- _end ----------------->|
         |                            |                                    |
         v                            |                                    v
+--------+----------------------------+------------------------------------+
|        | uncompressed kernel        | decompress code + compressed kernel|
+--------+----------------------------+------------------------------------+
0                                     ^                                    ^
                                      |                                    |
         +----------------------------+       +----------------------------+
         |                                    |
         |                                    |
         |                                    |
+--------+------------------------------------+----------------------------+
|        | decompress code + compressed kernel|                            |
+--------+------------------------------------+----------------------------+
0        ^
         |
     0x100000
```

通过compressed vmlinux的ld脚本 , 可以看出decompress code和compressed
kernel在内存中的分布 : 

```
+--------------+<- 0, _head
| .head.text   |
+--------------+<- _ehead
| .rodata..    |
| compressed   |
|              |
| (compressed- |
|  kernel)     |
+--------------+<- _text
| .text        |
+--------------+<- _rodata, _etext
| .rodata      |
+--------------+<- _got, _erodata
| .got         |
+--------------+<- _data, _egot
| .data        |
+--------------+<- _bss, _edata
| .bss         |
+--------------+<- _pgtable, _ebss
| .pgtable     |
| (x86_64 only)|
+--------------+<- _epgtable, _end
```

接着 , 调整并加载64bit的gdt : 

```
/* Load new GDT with the 64bit segments using 32bit descriptor */
addl	%ebp, gdt+2(%ebp)
lgdt	gdt(%ebp)
...
	.data
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x0000000000000000	/* NULL descriptor */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

打开PAE : 

```
/* Enable PAE mode */
movl	%cr4, %eax
orl	$X86_CR4_PAE, %eax
movl	%eax, %cr4
```

下面便是重新建立临时的页表 , 用于后续进入long mode . 

```
 /*
  * Build early 4G boot pagetable
  */
	/* Initialize Page tables to 0 */
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl

	/* Build Level 4 */
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)

	/* Build Level 3 */
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:	movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b

	/* Build Level 2 */
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:	movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b

	/* Enable the boot page tables */
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

由于这里建立的0-4G对应的地址空间 , 所以这里需要1个PML4E (4G/(1<<39)),
4个pdpe(4G/(1<<30)), 2048个pde(4G/(1<<21)), 每个物理页大小为2M , 
所以这里需要1个pgdt+1个pudt+4个(2048/512)个pmdt, 而每个table包含512 (1<<9)
个表项 , 每个表项为8字节 , 所以总共需要24kb(8*512*6)的空间 , 
当然从pgtable的定义中也可以得到验证 : 

```
.section ".pgtable","a",@nobits
.balign 4096
pgtable:
.fill BOOT_PGT_SIZE, 1, 0

# define BOOT_INIT_PGT_SIZE	(6*4096)
```

最终 , 4级页表的结构如下 : 

```
  pgtable+6*4096 +---------+
                 |pmde 2047|
                 +---------+
                 | pmdt-   |
                 | entry   |
                 | ...     |
                 +---------+
                 |pmde 1536|
  pgtable+5*4096 +---------+<----------+
                 |pmde 1535|           |
                 +---------+           |
                 | pmdt-   |           |
                 | entry   |           |
                 | ...     |           |
                 +---------+           |
                 |pmde 1024|           |
  pgtable+4*4096 +---------+<-------+  |
                 |pmde 1023|        |  |
                 +---------+        |  |
                 | pmdt-   |        |  |
                 | entry   |        |  |
                 | ...     |        |  |
                 +---------+        |  |
                 |pmde 512 |        |  |
  pgtable+3*4096 +---------+<----+  |  |
                 |pmde 511 |     |  |  |
                 +---------+     |  |  |
                 | pmdt-   |     |  |  |
                 | entry   |     |  |  |
                 | ...     |     |  |  |
                 +---------+     |  |  |
                 | pmde 1  |     |  |  |
                 +---------+     |  |  |
                 | pmde 0  |     |  |  |
  pgtable+2*4096 +---------+<-+  |  |  |
                 | pudt-   |  |  |  |  |
                 | entry   |  |  |  |  |
                 | ...     |  |  |  |  |
                 +---------+  |  |  |  |
                 | pude 3  +--+--+--+--+
                 +---------+  |  |  |
                 | pude 2  +--+--+--+
                 +---------+  |  |
                 | pude 1  +--+--+
                 +---------+  |
                 | pude 0  +--+
    pgtable+4096 +---------+<---+
                 | pgdt-   |    |
                 | entry   |    |
                 | ...     |    |
                 +---------+    |
                 | pgde 0  +----+
       pgtable   +---------+
```

最后 , enable long mode : 

```
/* Enable Long mode in EFER (Extended Feature Enable Register) */
movl	$MSR_EFER, %ecx
rdmsr
btsl	$_EFER_LME, %eax
wrmsr
```

并进入long mode的首个指令 `startup_64` : 

```
pushl	$__KERNEL_CS
leal	startup_64(%ebp), %eax
/* Enter paged protected Mode, activating Long Mode */
movl	$(X86_CR0_PG | X86_CR0_PE), %eax /* Enable Paging and Protected mode */
movl	%eax, %cr0
/* Jump from 32bit compatibility mode into 64bit mode. */
lret
...
	.code64
	.org 0x200
ENTRY(startup_64)
```

## long mode

首先 , 将通用段寄存器清0 : 

```
/* Setup data segments. */
xorl	%eax, %eax
movl	%eax, %ds
movl	%eax, %es
movl	%eax, %ss
movl	%eax, %fs
movl	%eax, %gs
```

将eflags寄存器清 0 : 

```
/* Zero EFLAGS */
pushq	$0
popfq
```

接着 , 将compressed kernel和decompress code拷贝到relocation区域的尾部 : 

```
/*
 * Copy the compressed kernel to the end of our buffer
 * where decompression in place becomes safe.
 */
	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss /* - $startup_32 */, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
```

紧接着 , 跳入到目标区域继续执行后续的代码 : 

```
/*
 * Jump to the relocated address.
 */
	leaq	relocated(%rbx), %rax
	jmp	*%rax
...
	.text
relocated:
```

将bss段清0 : 

```
/*
 * Clear BSS (stack is currently empty)
 */
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

因为relocation , 这里还需要调整GOT段内的地址 : 

```
/*
 * Adjust our own GOT
 */
	leaq	_got(%rip), %rdx
	leaq	_egot(%rip), %rcx
1:
	cmpq	%rcx, %rdx
	jae	2f
	addq	%rbx, (%rdx)
	addq	$8, %rdx
	jmp	1b
```

一些准备就绪 , 下面就需要解压compressed kernel到我们指定的目标区域 , 
该工作由 `extract_kernel` 函数完成 : 

```
/*
 * Do the extraction, and jump to the new kernel..
 */
	pushq	%rsi			/* Save the real mode argument */
	movq	%rsi, %rdi		/* real mode address */
	leaq	boot_heap(%rip), %rsi	/* malloc area for uncompression */
	leaq	input_data(%rip), %rdx  /* input_data */
	movl	$z_input_len, %ecx	/* input_len */
	movq	%rbp, %r8		/* output target address */
	movq	$z_output_len, %r9	/* decompressed length, end of relocs */
	call	extract_kernel		/* returns kernel location in %rax */
	popq	%rsi
```

该函数的参数如下 : 

1. rmode: 指向保存的启动参数 .

2. heap: 指向可用的heap的地址 . 这里的heap是预先申请好的一段空间 : 

```
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
...
#ifdef CONFIG_KERNEL_BZIP2
# define BOOT_HEAP_SIZE		0x400000
#else /* !CONFIG_KERNEL_BZIP2 */
# define BOOT_HEAP_SIZE		 0x10000
#endif
```

3. input_data: 指向compressed kernel

4. input_len: compressed kernel的大小

5. output : 指向指定的解压地址

6. output_len : 指向uncompressed kernel的大小

这里的 `input_data` , `input_len` , `output_len` 的值是通过一个叫mkpiggy的
程序在编译时生成的 .

该程序接收的参数为经过压缩的kernel文件 ,
之后通过读取该文件的大小 便得到了compressed kernel的大小 , 
通过读取该文件的后4个字节 , 得到uncompressed kernel的大小 , 
通过 `.incbin` 将compressed kernel的内容包含进来 . 

该程序最终将上述结果输出到 `piggy.S` 中 , 下面是我这里生成的该文件 : 

```
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 1777261
.globl z_output_len
z_output_len = 9491880
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
input_data_end:
```

在 `extract_kernel` 中 , 首先会选择最终的解压地址 ,
当 `kaslr` 打开时 , 会在空闲的内存空间中随机寻找一块区域 , 
否则仍选择传入的output地址作为最终的解压地址 . 

```
...
choose_random_location((unsigned long)input_data, input_len,
			(unsigned long *)&output,
			max(output_len, kernel_total_size),
			&virt_addr);
```

最终 , 根据编译时选择的压缩格式调用具体的解压方法进行解压 : 

```
debug_putstr("\nDecompressing Linux... ");
__decompress(input_data, input_len, NULL, NULL, output, output_len,
		NULL, error);
```

当完成解压之后 , 还有两件事需要完成 : 

1. 由于uncompressed kernel 本身是一个elf文件 , 
所以这里需要将elf文件中的 loadable segment拷贝到具体的位置中 : 

```
static void parse_elf(void *output)
{
	...
	for (i = 0; i < ehdr.e_phnum; i++) {
		phdr = &phdrs[i];

		switch (phdr->p_type) {
		case PT_LOAD:
#ifdef CONFIG_RELOCATABLE
			dest = output;
			dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
#else
			dest = (void *)(phdr->p_paddr);
#endif
			memmove(dest, output + phdr->p_offset, phdr->p_filesz);
			break;
		default: /* Ignore other PT_* */ break;
		}
	}
	...
}
```

2. 更新追加在uncompressed尾部的relocation table : 

```
static void handle_relocations(void *output, unsigned long output_len,
			       unsigned long virt_addr)
{
	...
	/*
	 * Calculate the delta between where vmlinux was linked to load
	 * and where it was actually loaded.
	 */
	delta = min_addr - LOAD_PHYSICAL_ADDR;
	...
	for (reloc = output + output_len - sizeof(*reloc); *reloc; reloc--) {
		...
		*(uint32_t *)ptr += delta;
	}
}
```

最终 , 返回uncompressed kernel的首地址并跳转到那 : 

```
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
				  unsigned char *input_data,
				  unsigned long input_len,
				  unsigned char *output,
				  unsigned long output_len)
{
	...
	return output;
}

// header_64.S
	call	extract_kernel		/* returns kernel location in %rax */
/*
 * Jump to the decompressed kernel.
 */
	jmp	*%rax
```

FIN.
