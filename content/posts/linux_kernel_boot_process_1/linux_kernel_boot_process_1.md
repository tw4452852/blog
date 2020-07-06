---
categories:
- Understand Linux Kernel
date: 2017-02-23T10:00:00
tags:
- linux
- kernel
title: Linux kernel boot process - part 1
---

## 概述

这里主要针x86_64平台进行分析 , 
kernel的启动过程主要经历这样几个阶段 : 
real mode -> protect mode -> long mode
下面结合4.10的kernel代码进行分析 . 

## real mode

在real mode下 , 地址总线共20位， 所以其最大可访问的内存大小为1Mb . 
在pc上电之后 , 第一条指令的地址位 0xffff * 16 + 0xfff0 , 也就是0xfffffff0 .
一般来说 , 该指令指向bios的启动代码 ,
紧接着bios会查找启动分区 , 而启动分区的标志是其最后2个字节为 0x55aa ,
如果发现了该启动分区 , bios会将该分区拷贝到物理地址位0x7c00的位置 , 
拷贝大小为512字节 ( 即一个分区的大小 ) , 最后将跳转到0x7c00 ,
从而将控制权交给启动分区代码 . 

### boot sector

kernel本身并不支持自启动 , 而必须通过bootloader进行加载 , 
常见的bootloader包括grub , syslinux等等 . 
所以一般放在启动分区的是bootloader的启动代码 , 
而如果kernel被放在了启动分区 , 
通常情况下 , 我们会在屏幕上看到如下信息 : 

```
Use a boot loader."

Remove disk and press any key to reboot..."
bootsec_info_end OMIT
```

其实这里是kernel的启动分区代码打印出来的 , 
我们可以从kernel的setup的链接脚本看出 : 

```
// arch/x86/boot/setup.ld

...
SECTIONS
{
	. = 0;
	.bstext		: { *(.bstext) }
	.bsdata		: { *(.bsdata) }
...
```

下面我们来看看启动分区的代码 . 
首先是初始化相关段寄存器， 同时关闭中断

```
// arch/x86/boot/header.S

.code16
.section ".bstext", "ax"

.global bootsect_start
bootsect_start:
	# Normalize the start address
	ljmp	$BOOTSEG, $start2

start2:
	movw	%cs, %ax
	movw	%ax, %ds
	movw	%ax, %es
	movw	%ax, %ss
	xorw	%sp, %sp
	sti
```

接着 , 通过bios提供的[0x10](http://www.ctyme.com/intr/rb-0106.htm)中断
将每个字符显示在屏幕 , 也就是我们看到的 : 

```
	cld
	movw	$bugger_off_msg, %si

msg_loop:
	lodsb
	andb	%al, %al
	jz	bs_die
	movb	$0xe, %ah
	movw	$7, %bx
	int	$0x10
	jmp	msg_loop

...
	.section ".bsdata", "a"
bugger_off_msg:
	.ascii	"Use a boot loader.\r\n"
	.ascii	"\n"
	.ascii	"Remove disk and press any key to reboot...\r\n"
	.byte	0
```

最后 , 使用bios [0x16](http://www.ctyme.com/intr/rb-1754.htm)中断
接收用户的按键 , 
然后调用[0x19](http://www.ctyme.com/intr/int-19.htm)中断进行重启 . 
一般情况下 , 这是pc会重新进行启动 , 为了防止0x19中断失败 , 
最后强制调转到 0xfffffff0 , 也就时bios的第一条指令所在的位置.

```
bs_die:
	# Allow the user to press a key, then reboot
	xorw	%ax, %ax
	int	$0x16
	int	$0x19

	# int 0x19 should never return.  In case it does anyway,
	# invoke the BIOS reset code...
	ljmp	$0xf000,$0xfff0
```

### kernel header

真正的入口是 `_start` , 而它只是一个调转指令 : 

```
	.globl	_start
_start:
		# Explicitly enter this as bytes, or the assembler
		# tries to generate a 3-byte jump here, which causes
		# everything else to push off to the wrong offset.
		.byte	0xeb		# short (2-byte) jump
		.byte	start_of_setup-1f
1:
...
```

为什么这里又是一个调转呢 , 其实该位置是位于kernel header中 , 
kernel header是kernel在x86平台与bootloader进行信息交互的一种方式 , 
kernel会在该头部中写入一些信息供bootloader加载时使用 , 
例如kernel希望bootloader将其加载到什么位置 , kernel的版本信息， 等等 . 
bootloader同样会将加载的相关信息写入到该头部中 , 
提供给kernel在执行时使用 . 
该头部的相关介绍可以参考kernel document ( Documentation/x86/boot.txt ) 
而这里的调转 , 正是跳过该头部 , 来到真正的setup代码 ： 

```
...
# End of setup header #####################################################

	.section ".entrytext", "ax"
start_of_setup:
# Force %es = %ds
	movw	%ds, %ax
	movw	%ax, %es
	cld
```

这里首先使es段寄存器和数据段寄存器ds一致 , 
随后清除direction标记.

### setup stack

为了尽快进入c代码 , 需要将stack建立好 , 这里会根据ss段寄存器的值进行不同处理 : 

```
movw	%ss, %dx
cmpw	%ax, %dx	# %ds == %ss?
movw	%sp, %dx
je	2f		# -> assume %sp is reasonably set
...
2:	# Now %dx should point to the end of our stack space
	andw	$~3, %dx	# dword align (might as well...)
	jnz	3f
	movw	$0xfffc, %dx	# Make sure we're not zero
3:	movw	%ax, %ss
	movzwl	%dx, %esp	# Clear upper half of %esp
	sti			# Now we should have a working stack
```

如果ss与ds一致 , 那么使用当前的ss与sp寄存 ,
这里还会将sp进行4字节对其 , 如果其太小那么使用 0xfffc作为sp的值 . 

如果ss与ds不一致 , 那么我们认为sp的值是非法的 : 

```
# Invalid %ss, make up a new stack
movw	$_end, %dx
testb	$CAN_USE_HEAP, loadflags
jz	1f
movw	heap_end_ptr, %dx
1:	addw	$STACK_SIZE, %dx
jnc	2f
xorw	%dx, %dx	# Prevent wraparound
```

这里还会根据是否使用heap来调整stack的位置 , 
对于heap的使用是根据header中的loadflags来区分的 : 

```
Field name:	loadflags
Type:		modify (obligatory)
Offset/size:	0x211/1
Protocol:	2.00+
  This field is a bitmask.
  ...
  Bit 7 (write): CAN_USE_HEAP
	Set this bit to 1 to indicate that the value entered in the
	heap_end_ptr is valid.  If this field is clear, some setup code
	functionality will be disabled.
```

最终 , stack可能的位置有如下三种 : 

1. ss与ds一致 : 

```
+------------+<---- sp
|            |
|            |
|  stack     |
|            |
+------------+<---- _end
|            |
| setup code |
+------------+<---- ss
```

2. ss与ds不一致 , 不使用heap :

```
+------------+<---- sp = _end + STACK_SIZE
|            |
|            |
|  stack     |
|            |
+------------+<---- _end
|            |
| setup code |
+------------+<---- ss
```

3. ss与ds不一致 , 使用heap :

```
+------------+<---- sp = heap_end_ptr + STACK_SIZE
|            |
|    stack   |
+------------+<---- heap_end_ptr
|    heap    |
+------------+<---- _end
|            |
| setup code |
+------------+<---- ss
```


### setup

stack建立好之后 , 下面是统一cs和ds的段寄存器 : 

```
# We will have entered with %cs = %ds+0x20, normalize %cs so
# it is on par with the other segments.
	pushw	%ds
	pushw	$6f
	lretw
6:
```

接着检查setup尾部的标记 : 

```
# Check signature at end of setup
	cmpl	$0x5a5aaa55, setup_sig
	jne	setup_bad
```

然后将bss段清0 : 

```
# Zero the bss
	movw	$__bss_start, %di
	movw	$_end+3, %cx
	xorl	%eax, %eax
	subw	%di, %cx
	shrw	$2, %cx
	rep; stosl
```

最后就可以进入c代码了 : 

```
# Jump to C code (should not return)
	calll	main
```

进入c代码之后 , 首先是将kernel保存到内部的数据结构中 : 

```
/* First, copy the boot header into the "zeropage" */
copy_boot_params();
```

接着会初始化console和heap , 接着会检测cpu状态 , 检测内存以及设置video mode
等等 . 

这里我们来看看heap相关的操作 , 首先是heap的初始化 : 

```
char *HEAP = _end;
char *heap_end = _end;		/* Default end of heap = no heap */

static void init_heap(void)
{
	char *stack_end;

	if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
		asm("leal %P1(%%esp),%0"
		    : "=r" (stack_end) : "i" (-STACK_SIZE));

		heap_end = (char *)
			((size_t)boot_params.hdr.heap_end_ptr + 0x200);
		if (heap_end > stack_end)
			heap_end = stack_end;
	} else {
		/* Boot protocol 2.00 only, no heap available */
		puts("WARNING: Ancient bootloader, some functionality "
		     "may be limited!\n");
	}
}
```

其中 `HEAP` 指向开始 , 而 `heap_end` 指向结束 , 
为了防止与stack溢出 , 这里 `heap_end` 最大指向stack的结束位置 . 

同时 , 这里还提供了3个接口 , 一个用于查看当前heap是否有足够的指定的空间大小： 

```
static inline bool heap_free(size_t n)
{
	return (int)(heap_end-HEAP) >= (int)n;
}
```

一个用于从heap分配申请的大小 : 

```
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

还有一个用于释放当前heap上所有分配的空间 : 

```
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

有了以上三个接口 , 一般的用法为 : 

```
RESET_HEAP()
...
if (!heap_free(size))
	// no enough free space
	...
else
	GET_HEAP(struct_a, number_of_struct_a)
	...
```

接着kernel会设置一个空的idt以及一个包含寻址空间为0-4Gb的代码段和数据段的gdt ,

最终调用 `protected_mode_jump` 完成从real mode到protect mode的跳转 . 

```
...
protected_mode_jump(boot_params.hdr.code32_start,
		(u32)&boot_params + (ds() << 4));
```

该函数接受两个参数 : 

1. entrypoint(32bit): protect mode的首个指令的物理地址 ,
这里由bootloader指定 , 它保存在kernel header中 , 该参数保存在eax寄存器中 . 

2. bootparams(32bit): 保存启动参数的数据结构的物理地址 , 
该参数保存在edx寄存器中 . 

进入 `protected_mode_jump` 后 , 首先将启动的相关信息保存在esi寄存器中 , 
因为edx寄存器后面会被用到 . 

```
movl	%edx, %esi		# Pointer to boot_params table
```

接着调整label 2处的地址 : 

```
xorl	%ebx, %ebx
movw	%cs, %bx
shll	$4, %ebx
addl	%ebx, 2f
...
2:	.long	in_pm32			# offset
...
	.code32
	.section ".text32","ax"
GLOBAL(in_pm32)
```

label 2处最开始保存的是进入protect mode的首个指令的偏移 , 
这里加上cs段基地址就变成了其真实的物理地址 , 
这样就为后面的调转做好了准备 . 

接着 , 将ds段和tss段选择符分别保存在cx和di寄存器中 : 

```
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

然后就是打开cr0寄存器中的protect mode使能位 : 

```
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl	# Protected mode
movl	%edx, %cr0
```

最终 , 通过长调转进入protect mode : 

```
# Transition to 32-bit mode
.byte	0x66, 0xea		# ljmpl opcode
2:	.long	in_pm32			# offset
.word	__BOOT_CS		# segment
```

进入protect mode之后 , 首先便是重新设置各个段寄存器 , 同时调整esp寄存器 : 

```
# Set up data segments for flat 32-bit mode
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
# The 32-bit code sets up its own stack, but this way we do have
# a valid stack if some debugging hack wants to use it.
addl	%ebx, %esp
```

接着 , 将相关通用寄存器清0 : 

```
# Clear registers to allow for future extensions to the
# 32-bit boot protocol
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

最后 , 跳转到kernel protect mode的首个指令 , 该地址保存在eax中 : 

```
jmpl	*%eax			# Jump to the 32-bit entrypoint
```

FIN.
