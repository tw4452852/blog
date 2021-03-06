---
categories:
- Understand Linux Kernel
date: 2014-01-01T20:00:00
tags:
- linux
- kernel
- intel
title: Memory Addressing
---

## 简介

本文主要讲解内存编址相关的内容,主要包括2部分:

- 内存编址中的基本概念, 包括地址类型,不同类型的相互转换等等.
- linux kernel中对内存编址的支持.

## 地址类型

地址的类型,从用户态的程序到最终发送到地址总线上,依次分为3类:

- 逻辑地址: 这也就是程序运行时所用的地址,该地址既可以指向一条指令,也可以指向数据.这种类型的主要存在于x86平台,它要求程序员将程序分割成不同的段(数据段,代码段...).逻辑地址由段指示符和偏移两部分组成(最终地址=段开始地址+偏移).
- 线性地址(又称虚拟地址): 一个32bit的无符号整数,地址范围0-4G.
- 物理地址: 指向真实的内存单元,和地址总线上的管脚一一对应,通常可以用32位或36位无符号整数表示.

![translation](translation.png)

内存管理单元(mmu)通过一个叫段管理单元的硬件电路将逻辑地址装换成对应的线性地址,
接着,通过第二个硬件电路(页管理单元),将线性地址转换成对应的物理地址,
最终发送到地址总线上,从而完成最终的存储单元的寻址.

### 段管理单元

首先,我们来说段管理单元,之前我们说过逻辑是由一个段指示符和偏移组成.
这里的段指示符更准确的说是一个段选择子,那么这个选择子,肯定是选择我们到底用的哪个段了,
那么在哪里选择呢?

这里就要说到全局描述符表(GDT)和局部描述符表(LDT),
顾名思义,GDT只有一份,所有程序共享,而LDT每个程序都可以有自己的.
无论是GDT还是LDT,表中都是一个个的段描述符.

这里不具体讲解表项中的每个字段的含义,不过我们只需知道里面包含一些段的基本描述信息:
段开始地址,段长,段类型等等.

下面列举几个常见的段描述符:

![segment_desc](segment_desc.png)

PS:
为什么段开始地址和段长度支离破碎,这个是有历史的原因的,简单的来说是为兼容性的考虑,
可以想象,最开始的段描述符没有这么复杂.

在系统初始化的过程中,kernel会通过特定的指令(lgdt/lldt)告知段管理单元GDT/LDT在内存中的地址,
这样,如果我们选择了哪个段,硬件就会加载那个段的描述符.

那么我们前面所说的选择子其实就是GDT/LDT的index,由于每个段描述符8个字节,
这样通过GDT/LDT的开始地址+选择子*8=所选的段描述符的地址.

有了段描述符,那么从中获得段开始地址+32bit的逻辑地址偏移=线性地址.

![logical_translation](logical_translation.png)

这里的TI(table indicator)表示在GDT(TI=0)还是在LDT(TI=1)中查找描述符.

#### 段保护机制

段管理单元提供基于段的保护机制,下面我们来看看具体是如何实现的.

首先要说的是在每个段描述符中都有2bit的描述符权限等级(DPL),
该项设定访问该段时CPU至少应该处于什么权限等级
(CPU有4个权限等级,从高到低为0-3).

例如,当一个描述符中的DPL=0,那么只有当CPU权限为0级(最高)时,
才能够访问该段.而如果描述符中的DPL=3,那么无论CPU处于哪个等级(3/2/1/0)
都可以访问该段.

那么CPU等级是如何改变的呢?
其实每个CPU都有几个段寄存器(代码段(cs),数据段(ds),堆栈段(ss)等等),
其中每个段寄存器中存放的都是我们之前说的段选择子.
之前我们还没有仔细看过段选择子的格式,只知道里面有个index和TI,
下面便是段选择子的格式:

![segment_selector](segment_selector.png)

可以看到每个段选择子都有2bit的RPL,大家可能已经知道这就是制定CPU权限等级的地方.
准确的说,只有当代码段(cs)改变时,CPU才会将当前的权限等级置为代码段选择子中的RPL.
这样只是加载一个RPL=3的代码段,是不能对DPL=0的段进行访问的.

### Linux kernel对段管理的支持

在kernel中,对段管理的支持非常简单.
首先,kernel并没有使用全部的4个CPU权限等级,而只是使用其中的2个
(0级:kernel mode, 3级:user mode).
其次,在2个mode下,分别有代码段和数据段,
在代码中分别用 `__KERNEL_CS` , `__KERNEL_DS` , `__USER_CS` , `__USER_DS` ),
下面我们来看下这4个描述符中的内容:

![linux_segments](linux_segments.png)

可以看出无论的是kernel mode还是user mode逻辑地址和线性地址是一一对应的,
唯一不同的段的DPL分别被置为0和3.

那么为什么linux kernel没有充分利用的段管理单元提供的映射功能呢,
这里有两方面的考虑:

- linux kernel具有通用性,适用于不同的平台,而在大多数平台上,都不存段管理单元,而只有x86平台才有,所以为了统一,linux在x86平台上将逻辑地址和线性地址做了一一对应的映射.
- 段管理单元和后面要说的页管理单元,在保护功能上有一定的重复,相对于页保护,段保护的粒度更大,所以linux着重了对页保护功能的支持.

### 页管理单元

现在有了线性地址,下一步就是将其转换成对应的物理地址,这部分工作由页管理来完成.

对于页管理单元而言,真实的物理单位是由一个个的物理页组成,每个物理页的大小一般为4Kb
(当然也有例外,后面会有说明).
这样,一个线性地址会首先找到它对应的物理的页,然后将物理页的起始地址+对应的偏移,便得到最终的物理地址.

那么一个线性地址是如何找到其对应的物理页的呢,
这里页管理单元通过2个步骤来完成,
首先是找到该地址对应的页目录项,其中存放对应的页表的物理地址.
接着再在对应的页表中,查找对应的页表项,其中存放的就是该物理页的起始地址:

![paging_x86](paging_x86.png)

可以看出这是一个2级查找,为什么要使用这种方式呢,原因是节省内存的使用,
因为有个程序运行,一般不会完全使用的4G的逻辑地址空间,而只是使用其中的一小部分,
那么我们只需要为其使用的那部分分配页表即可.

这里需要注意的是,每个程序都有自己的地址空间,换句话说,它们都有自己的页目录,
每当进程切换时,kernel都会将即将换出的程序的页目录的地址保存在其自己的程序描述符中,
再将即将换入的程序的页目录的地址填入到cr3寄存器中,完成程序地址空间的切换.

上面我们看到每个物理页的大小是4Kb,当然,现在的cpu也支持更大的物理页(4Mb),
这样转换关系就从之前的2级变为了1级:

![paging_x86_ext](paging_x86_ext.png)

那么到底一个物理页是4Kb还是4MB是由页目录项的PS(Page
Size)来决定,PS=1,那么对应的物理页为大页,反之则为小页.
这里需要注意,大页和小页是可以共存的.

时代在进步,上面我们所说的物理地址都是32bit,也就是说其最多可以使用4Gb的memory,
那么如果真实的memory的容量大于4Gb呢,intel为了支持更大的内存容量,
将其地址总线由32bit增大为36bit,这样就可以支持最多64Gb的memory,
该技术叫物理地址扩展(PAE).

当PAE使能之后,物理地址从32bit变为了36bit,那么对应的页表项由之前的
2^20个变为了2^24个,那么之前32bit的页表项也不能满足要求
(24bit的物理地址+12bit的flag > 32bit),所以intel将其页表项的大小从之前的4字节调整到8字节.
这样一个4k的页表只能包含512个表项而不是之前的1024个.

与此同时,一个新的概率页目录表(PDPT)被加入进来,其中每个表项为8个字节,
分别指向一个页目录.

这样cr3寄存中存放的不再是页目录的地址,而是页目录表的地址.

那么一个线性地址被分为了下面四个部分:

- bit30-31: 页目录表(PDPT[4])中的index
- bit21-29: 页目录(PD[512])中的index
- bit12-20: 页表(PT[512])中的index
- bit0-11: 在4Kb物理页中的偏移

当支持大页时,一个页的大小也不再是4MB而是2MB:

- bit30-31: 页目录表(PDPT[4])中的index
- bit21-29: 页目录(PD[512])中的index
- bit0-20: 在2MB物理页的偏移

以上都是在32位架构下的,下面这张图是64位架构下不同平台是如何编址的:

![paging_64](paging_64.png)

### 页保护机制

首先每个页目录项和页表项中都有User/Supervisor的flag,当该flag为0,
那么该表项只能当cpu权限等级小于3级时才能访问.

同时,还有个Read/Write控制表项的访问方式,当该标志位0时,表明该表项为只读模式,
而为1时,表明该表项可读可写.

### Linux kernel对页管理的支持

通过上述的分析,我们可以看出,对于PAE是否开启,是否支持大小页,以及是否是32位还是64位架构,
都会影响最终的转换方式.
linux kernel为了做到统一以及兼容性的考虑,使用如下的转换方式:

![paging_linux](paging_linux.png)

其中在某些情况,图中的某些部分是不存在的(也就是其在线性地址中占据的位数为0),
例如当处于32位架构下,同时PAE disable以及使用小页的情况下,
upper dir和middle dir是不存在的.

关于页管理单元,还有很多内容没有讲解,比如缺页异常,cache相关,TLB相关,
后续的文章会对其进行详细的介绍.

FIN.
