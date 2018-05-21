# From Power on to kernel boot

## 1st instruction after Power-up
PC 在 power on 或者 reset(assertion of the RESET# pin) 后，系统总线上的 CPU 会做处理器的硬件初始化，这包括：将所有寄存器复位成默认值，将 CPU 置于 real mode，invalidate 内部所有的 cache，比如 TLB 等；然后对于 SMP 系统来说，会执行硬件的 multiple processor (MP) initialization protocol 来选择一个 CPU 成为 BSP(bootstrap processor)，被选出的 BSP 则立刻从当前的 CS:EIP 中取指令执行。

Power-up 后，寄存器 CR0 的值为 0x60000010，意味着将 CPU 置于关闭 paging 的 real mode 状态.
![CR0_AFTER_PowerUp](cr0_after_powerup.png)

其他的寄存器复位后的状态如下图示:
![Registers After PowerUp](register_after_powerup.png)

此处需提前介绍一下 X86 的 segment register，如下图：

![Segment Register](segment_register.png)

每个 segment register 包含了一个 visual part 和一个 hidden part(有时也会叫做“descriptor cache” or  ”shadow register“)，当 segment selector 被加载进 visual part 时，处理器也会自动将相应 segment descripter 中的 Base Address, Limit, Accesstion Information 加载进 hidden part，这些 cache 在 segment register 中的信息，使得处理器做地址翻译时不必从 segment descriptor 中读取 base address 等信息，从而省去了不必要的 bus cycle。如果 segment descriptor table 发生了变化，处理器需要显式的重新加载 segment register，不然的话，地址翻译还是使用老的 segment descriptor 中的信息。也就是说，当使用 CS:EIP 的方式去寻址时(或者说在计算 linear address 时)，实际是使用 hidden part 中的 Base Address + EIP value。

上面已知 bootstrap processer 会从 CS：EIP 中取出第一条指令来执行，Power-up 后，CS.Base = FFFF0000H, EIP=0000FFF0H，所以第一条指令的地址 = FFFF0000H + 0000FFFFH = FFFFFFF0H，这超出了 real mode 最大 1M 的寻址空间，但 PC 复位后寄存器的值就是那样。PC 复位后第一次加载新的值进 CS 后，接下来的 linear address 的翻译方式就是我们知道的： selector value << 4 + EIP。

主板上的 chipset 会把第一条指令的地址 FFFFFFF0H 映射到包含 bios 的 rom 中，典型的第一条指令是：

    FFFFFFF0:    EA 5B E0 00 F0         jmp far ptr F000:E05B

这个跳转指令就会刷新 CS 的值，那么 CS.Base 的值就变成了 F0000H(F000H << 4)，接下来的寻址就是按照 real mode 的方式： CS selector << 4 + EIP。

BIOS 执行到最后是从已设置的启动设备中读取指令到内存，并跳转过去执行。我们以安装了 grub2 的硬盘启动为例说明。

## GRUB2

BIOS 将硬盘 MBR 中的内容加载到 0000:7C00 并跳转到这个地址继续执行。为什么将 boot sector(MBR) 加载到这个比 32kb 小 1024 byte 的地址？ 这里有[一篇科普](http://www.ruanyifeng.com/blog/2015/09/0x7c00.html)。

接下来我们分析 grub2 是如何执行并将 OS 加载到内存执行的。这里有一篇很好的材料讲述了 [GRUB2 的启动过程](https://www.slideshare.net/MikeWang45/grub2-booting-process)

grub2 的启动分为几个阶段：stage1，stage2，文件系统的剩余部分。stage1 是执行 MBR 中的代码，也就是我们常说的 boot sector； stage2 执行的代码存在 MBR 之后，第一个磁盘分区之前(以前这个 size 有 63 sector，今天普遍是 2048 个 sector)，由 boot sector 的代码加载到内存中执行，这里面包含了访问文件系统的驱动程序，可以从文件系统读取 stage2 的内容；stage2 包含读取文件系统上的 grub 的配置文件和其他模块等。所以，grub 的代码 image 被分为几个部分安装在硬盘中.

具体来说，grub2 的代码 image 分为 3 个部分： MBR 中的 boot.img，stage2 区域的 core.img，文件系统中的模块。安装在硬盘上的 grub2 的布局如下图：

![grub](grub_images.png)

core.img 又包含了多个 image 和模块，它的布局如下图：
![grub core image](core_image.png)

boot.img 的功能是将 core.img 的第一个 sector 的内容(也即 diskboot.img)加载到内存执行，core.img 中剩余的部分由它的 diskboot.img 完成。

### MBR/boot.img

boot.img 对应的 grub2 的源码文件是 grub-core/boot/i386/pc/boot.S。这里有一篇对[boot.img的完整分析](https://www.funtoo.org/Boot_image)，只是代码比较老，但整体逻辑是一样的

文件开始部分定义了两个宏，可以略过不看

	.macro floppy
	xxx
	.endm
	.macro scratch
	xxx
	.endm

boot sector 开始的部分为了兼容 FAT/HPFS BIOS parameter block(BPB) 预留了空间，前 12 bytes(0xB) 由 .marco scratch 实现。BPB 是一个存储在 VBR(volumn boot record) 中用来描述磁盘或者分区的物理布局的数据结构。在[这篇介绍](https://en.wikipedia.org/wiki/BIOS_parameter_block)的表：Format of full DOS 7.1 Extended BIOS Parameter Block (79 bytes) for FAT32 中，可以看出，BPB 起始位置是 boot sector 的 offset 0xB 处，也即12 bytes，正好是 .macro scratch 定义的大小；整个 BPB 的长度是 0x52 + 0x8 = 0x5a，正好是 grub2 代码中宏 *GRUB_BOOT_MACHINE_BPB_END* 的值。

接下来是一堆参数定义，有些字段是需要在安装 grub 的时候被写入的

	LOCAL(kernel_address):
	.word	GRUB_BOOT_MACHINE_KERNEL_ADDR

	#ifndef HYBRID_BOOT
	.org GRUB_BOOT_MACHINE_KERNEL_SECTOR
	LOCAL(kernel_sector):
	.long	1
	LOCAL(kernel_sector_high):
	.long	0
	#endif

	.org GRUB_BOOT_MACHINE_BOOT_DRIVE
	boot_drive:
	.byte 0xff	/* the disk to load kernel from */
			/* 0xff means use the boot drive */

注意，在 grub 的上下文中，kernel 指的是 core.img。
*kernel_sector* & *kernel_sector_high* 记录 core.img 在磁盘上的第一个扇区号，用于记载它时使用;*kernel_address* 表示 core.img 第一个扇区被加载到内存中的地址，由宏 *GRUB_BOOT_MACHINE_KERNEL_ADDR*定义，
在 i386 PC 上，这个地址是 0x8000，代码如下：

	/* The address where the kernel is loaded.  */
	#define GRUB_BOOT_MACHINE_KERNEL_ADDR	(GRUB_BOOT_MACHINE_KERNEL_SEG << 4)

	#define GRUB_BOOT_MACHINE_KERNEL_SEG GRUB_OFFSETS_CONCAT (GRUB_BOOT_, GRUB_MACHINE, _KERNEL_SEG)

	#define GRUB_OFFSETS_CONCAT_(a,b,c) a ## b ## c
	#define GRUB_OFFSETS_CONCAT(a,b,c) GRUB_OFFSETS_CONCAT_(a,b,c)

	/* The segment where the kernel is loaded.  */
	#define GRUB_BOOT_I386_PC_KERNEL_SEG	0x800

单纯看代码肯定无法看出为什么这个宏定义为 0x8000。在编译完代码后，发现 grub-core/Makefile 会发现有这么一行:

	TARGET_CPPFLAGS =  -Wall -W  -DGRUB_MACHINE_PCBIOS=1 -DGRUB_MACHINE=I386_PC -m32 bluhbluh...

这样就明白了。

真正有用的代码开始于 lable: real_start。通过调用 INT 0x13 判断磁盘是否支持 LBA 访问模式，不然就使用传统的 CHS 模式。以 LBA 为例，继续调用 INT 0x13，从 core.img 的第一个 sector 的起始位置读入一个 block(size 是多少？) 到地址为 *GRUB_BOOT_MACHINE_BUFFER_SEG*(0x7000):0 的 buffer 中，关于这个 INT 0x13 调用的详细解释参考[这篇](http://www.ctyme.com/intr/rb-0708.htm)。然后调用函数 *copy_buffer*，从源地址 *GRUB_BOOT_MACHINE_BUFFER_SEG*:0 拷贝 512 bytes 到 0:*GRUB_BOOT_MACHINE_KERNEL_ADDR*，也即从 0x7000:0 拷贝到 0:0x8000，然后 jmp 到 *GRUB_BOOT_MACHINE_KERNEL_ADDR*。

MBR 中的 boot.img 的工作就完成了。

### diskboot.img

由 core.img 的图示可知，它的第一个 sector 的内容是 diskboot.img。diskboot.img 对应的源代码文件是 grub-core/boot/i386/pc/diskboot.S。diskboot.img 的执行环境，也即寄存器，由 boot.img 设置，此时的环境如下：

1. 有可用的堆栈(SS 和 SP 已配置)。
2. 寄存器 DL 中保存正确的引导驱动器。
3. 寄存器 SI 保存着 DAP(Disk Address Packet) 的地址，DAP 定义在 boot.S 中，由 .macro scratch 实现。

diskboot.img 的工作是将 core.img 中剩余的部分继续加载到内存，并跳转过去执行。diskboot.img 的工作本质上和 boot.img 一样，都是借助 BIOS 的 interrupt service 读取磁盘 sector 的内容到内存，只不过 diskboot.img 需要加载多个 sector 而已。

diskboot.img 需要知道 core.img 剩余部分所在的 sector，显然，这是安装 grub 的时候才会知道，grub 的安装程序将 core.img 占据的 sector 的信息写入 diskboot.img。这部分空间在 diskboot.S 的尾部：

	.org 0x200 - GRUB_BOOT_MACHINE_LIST_SIZE
	LOCAL(firstlist):	/* this label has to be before the first list entry!!! */
				/* fill the first data listing with the default */
	blocklist_default_start:
	/* this is the sector start parameter, in logical sectors from
	   the start of the disk, sector 0 */
	.long 2, 0
	blocklist_default_len:
	/* this is the number of sectors to read.  grub-mkimage
	   will fill this up */
	.word 0
	blocklist_default_seg:
	/* this is the segment of the starting address to load the data into */
	.word (GRUB_BOOT_MACHINE_KERNEL_SEG + 0x20)

对应了 grub 中的数据结构：

	struct grub_pc_bios_boot_blocklist
	{
	  grub_uint64_t start;
	  grub_uint16_t len;
	  grub_uint16_t segment;
	} GRUB_PACKED;

为什么这段空间被标以 label: firstlist 呢？一个 blocklist 描述一段连续的磁盘区域，而在某些情况下，core.img 有可能被分成多块安装在磁盘上，所以可能存在多个 blocklist，如果有多个的时候，这段空间会紧挨着 firstlist 向 diskboot.img 开始的方向延伸。

读取 sector 结束，通过下面的代码跳转到 0:0x8200 开始运行

	LOCAL(bootit):
	/* print a newline */
	MSG(notification_done)
	popw	%dx	/* this makes sure %dl is our "boot" drive */
	ljmp	$0, $(GRUB_BOOT_MACHINE_KERNEL_ADDR + 0x200)

跳转后的代码是 lzma_decompress.img 的内容。

### lzma_decompress.img

lzma_decompress.img 对应的源码是 grub-core/boot/i386/pc/startup_raw.S，此文件中又 include 同目录下的  "lzma_decode.S"，这是 lzma 的算法核心。它的工作是解压缩它后面的压缩代码，并跳转过去，由 core.img 的图示可知，跳转到 kernel.img，由名字可知，这是 grub 的核心代码，它对应的代码在 grub-core/kern 目录下。从某种意义上说，kernel.img 的代码才是 grub 真正的开始。

startup_raw.S 的开头部分是一条跳转指令：

	ljmp $0, $ABS(LOCAL (codestart))

是为了跳过开头部分的数据区域，跳到真正的代码处：codestart。这块 special data area，看其名字：*GRUB_DECOMPRESSOR_MACHINE_COMPRESSED_SIZE*，*GRUB_DECOMPRESSOR_MACHINE_UNCOMPRESSED_SIZE*，应该是生成这些 image 时，由 grub 将数据填写到此处。

将紧挨着 lzma_decompress.img 的数据(开始于 decompressor_end)解压缩到临时解压缩区域 *GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR* 处，并跳转过去执行，代码如下：

	post_reed_solomon:

	#ifdef ENABLE_LZMA
		movl	$GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR, %edi
	#ifdef __APPLE__
		movl	$decompressor_end, %esi
	#else
		movl	$LOCAL(decompressor_end), %esi
	#endif
		pushl	%edi
		movl	LOCAL (uncompressed_size), %ecx
		leal	(%edi, %ecx), %ebx
	/* Don't remove this push: it's an argument.  */
		push 	%ecx
		call	_LzmaDecodeA
		pop	%ecx
		/* _LzmaDecodeA clears DF, so no need to run cld */
	popl	%esi
	#endif

	movl	LOCAL(boot_dev), %edx
	movl	$prot_to_real, %edi
	movl	$real_to_prot, %ecx
	movl	$LOCAL(realidt), %eax
	jmp	*%esi

%edi 被赋值为 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR， 然后被 push 到 stack 上保存，调用 _LzmaDcodeA 函数后又 popl %esi，最后 jmp *%esi，也即 jmp 到 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR。

### 安装 GRUB

grub-mkimage 的作用是生成 core.img，虽然它的 man page 没有明确说明。一般情况下，core.img 是 grub-install 自动创建的，所以不需要手动调用 grub-mkimage，但是不代表不可以。
由上面的图示可知 core.img = diskboot.img[1] + lzma_decompress.img[2] + kernel.img + <some mods>

1. diskboot.img 用于硬盘启动的环境下，对于其他的情况，比如 CDROM，网络启动，这个 image 可以是 cdboot.img，pxeboot.img

2. 由于 kernel.img + <mods> 是会被压缩的，所以必须有相应的解压缩代码，不同的压缩算法对应了不同的 decompress image，对于 x86，默认是 lzma。