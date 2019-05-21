# From Power on to linux kernel

## 1st instruction after Power-up
PC 在 power on 或者 reset(assertion of the RESET# pin) 后，系统总线上的 CPU 会做处理器的硬件初始化，这包括：将所有寄存器复位成默认值，将 CPU 置于 real mode，invalidate 内部所有 cache，如 TLB 等；对于 SMP 系统，还会执行硬件的 multiple processor (MP) initialization protocol 来选择一个 CPU 成为 bootstrap  processor(BSP)，被选出的 BSP 则立刻从当前的 CS:EIP 中取指令执行。

Power-up 后，寄存器 CR0 的值为 0x60000010，意味着将 CPU 置于关闭 paging 的 real mode 状态.
![CR0_AFTER_PowerUp](cr0_after_powerup.png)

其他的寄存器复位后的状态如下图示:
![Registers After PowerUp](register_after_powerup.png)

此处需提前介绍一下 X86 的 segment register，如下图：

![Segment Register](segment_register.png)

segment register 包含 visual part 和 hidden part(有时也叫做 descriptor cache 或 shadow register)，当 segment selector 被加载进 visual part 时，处理器会自动将相应 segment descriptor 中的 Base Address, Limit, Accesstion Information 加载进 hidden part。这些 cache 在 segment register 中的信息，使得处理器做地址翻译时不必从 segment descriptor 中读取 base address 等信息，从而省去了不必要的 bus cycle。如果 segment descriptor table 发生了变化，处理器需要显式的重新加载 segment register，否则，地址翻译仍使用老的 segment descriptor 中的信息。也就是说，当使用 CS:EIP 的方式去寻址时(或者说在计算 linear address 时)，实际是使用 hidden part 中的 Base Address + EIP value。

由上文知， BSP 从 CS：EIP 中取第一条指令来执行。Power-up 后，CS.Base = FFFF0000H, EIP=0000FFF0H，所以第一条指令的地址 = FFFF0000H + 0000FFFFH = FFFFFFF0H，这个地址被映射到 ROM(ROM 是一种古老存储技术，用于存储 firmware，但这种技术不断更新。为避免困扰，本文仍然通称为 ROM) 上的 BIOS 中的某条指令。当 PC 复位后第一次加载新的值到 CS 后，接下来的 linear address 的翻译方式就是我们知道的： selector value << 4 + IP。

主板上的 chipset 把第一条指令的地址 FFFFFFF0H 映射到包含 BIOS 的 ROM 中，典型的第一条指令是：

    FFFFFFF0:    EA 5B E0 00 F0         jmp far ptr F000:E05B

这个跳转指令会刷新 CS 的值，那么 CS.Base 的值就变成了 F0000H(F000H << 4)，接下来的寻址就是按照 real mode 的方式： CS selector << 4 + EIP。

BIOS 第一条指令的具体细节很可能在不同的芯片/平台上而不同，BIOS 代码一般不是开源的，况且近年来出现的新的 firmware: EFI，就目前网络上各种资料看下来，笔者还无法得出一个 general 的答案。基于上面的典型第一条指令，可以推理一种执行过程：第一条指令跳转后，下面的地址都在1M范围内了。以 Q35 chipset 举例，chipset 的 Programmable Attribute Map(PAM) 寄存器会控制 768 KB 到 1 MB 地址空间中的 13 个 sections 的访问属性，开机后这些寄存器默认值的行为是 DRAM Disabled: All accesses are directed to DMI，也就是说对上述地址范围的访问会被 route 到 4G 的最后 1M 空间内，也就是说执行的代码还是来自 ROM 中的BIOS。后续 BIOS 是否做 memory shadowing(将自己 copy 到第 1M 地址空间的 DRAM)都可，只是 access 的速度有差别。另一方面，传统的 BIOS 都是运行在 real mode 下，它能看到的地址空间只有第 1M。 此外，所谓的带外管理技术(如 Intel 的 Management Engine)，也可能提前将 BIOS copy 到第 1M 的 DRAM 空间中。

BIOS 执行到最后是从已设置的启动设备中加载第一个 sector 的内容到内存，并跳转过去执行。我们 grub2 的硬盘启动为例分析。

## GRUB2 booting process

本节以只有一块硬盘的 PC 为例，基于 grub2 的代码分析 grub 的工作流程。这篇材料：[grub2-booting-process](https://www.slideshare.net/MikeWang45/grub2-booting-process) 可以作为很好的参考。

BIOS 的最后工作是将硬盘 MBR(也叫 boot sector) 中的内容加载到地址 0000:7C00，并跳转过去继续执行。为什么将 MBR 加载到这个比 32kb 小 1024 byte 的地址？ 这里有[一篇科普](http://www.ruanyifeng.com/blog/2015/09/0x7c00.html)。使用 MBR 这个术语是为了和 VBR 区分，Master boot record 是整个硬盘的第一个 sector，Volume boot record 是分区的第一个 sector。

很显然，仅仅 512 bytes 大小的 boot sector 是装不下一个功能强大的 grub 的，所以 grub 使用的方案是分成几个部分，第一部分(也是最小的)被安装在 MBR 中，其他的较大的部分放在别的位置，被 MBR 加载。用 grub 的术语讲，grub 的工作流程被分为几个 stage：stage 1，stage 1.5，stage 2。对于 grub2 来说：

* stage 1 是位于 MBR 中的 boot.img，用于加载 stage 1.5 的 core.img。(boot.img 在某些场景下也可以安装在 VBR 中)；
* stage 1.5 是位于 MBR 和第一个磁盘分区之间(以前这个 size 有 63 个 sector，今天普遍是 2048 个 sector)的 core.img，它包含了访问文件系统的驱动程序，可以从文件系统读取 stage 2 的内容；
* stage 2 是位于 /boot/grub 目录下的所有文件，包括配置文件和模块等。

安装在硬盘上的 grub2 的布局如下图：

![grub](GNU_GRUB_on_MBR_partitioned_hard_disk_drives.png)

简化版长这样：
![grub](grub_hdd_layout.png)

core.img 又包含了多个 image 和模块，它的布局如下：
![grub core image](core_image.png)

### boot.img/MBR/boot sector
boot.img 仅仅将 core.img 的第一个 sector 的内容(即 diskboot.img)加载到内存执行，core.img 中剩余的部分由它的 diskboot.img 继续加载到内存。boot.img 对应的 grub2 的源码文件是 grub-core/boot/i386/pc/boot.S。

对于 boot.S 的完整分析可以参考：

1. [Boot image](https://www.funtoo.org/Boot_image):基于较老的代码，但整体逻辑一样。
2. [boot.S & diskboot.S 分析](https://blog.csdn.net/conansonic/article/details/78482766):基于最新的代码(2018/4)

文件开头定义了两个宏:

	.macro floppy
	xxx
	.endm
	.macro scratch
	xxx
	.endm

其中, .macro scratch 只是声明了一些变量空间，用于下面的代码使用 BIOS INT 13h 时使用。

boot.s 的开头为了兼容 FAT/HPFS BIOS parameter block(BPB) 预留了空间，BPB 是一个存储在 VBR(volumn boot record) 中用来描述磁盘或者分区的物理布局的数据结构。BPB 空间对于 MBR 来说是不必要的，但某些场景下 grub 使用同一个 boot.img 安装到 VBR 中，VBR 是可能需要 BPB 的，所以需要预留这部分空间。在[这篇介绍](https://en.wikipedia.org/wiki/BIOS_parameter_block)的表：Format of full DOS 7.1 Extended BIOS Parameter Block (79 bytes) for FAT32 中，可以看出，BPB 开始于 boot sector 的 offset 0xB 处，BPB 的 size 是 0x47 + 0x8 = 0x4F，0xB + 0x 4F = 0x5A，正是宏 *GRUB_BOOT_MACHINE_BPB_END* 的值。

接下来是一堆参数定义，需要在安装 grub 的时候被写入(除了GRUB_BOOT_MACHINE_KERNEL_ADDR)：

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

在 grub 的上下文中，kernel 指的是 core.img。*kernel_sector* & *kernel_sector_high* 记录 core.img 在磁盘上的第一个扇区号; *kernel_address* 表示 core.img 第一个扇区被加载到内存中的地址，由宏  *GRUB_BOOT_MACHINE_KERNEL_ADDR* 定义，在 i386 PC 上，这个宏的值是 0x8000，代码如下：

	/* The address where the kernel is loaded.  */
	#define GRUB_BOOT_MACHINE_KERNEL_ADDR	(GRUB_BOOT_MACHINE_KERNEL_SEG << 4)

	#define GRUB_BOOT_MACHINE_KERNEL_SEG GRUB_OFFSETS_CONCAT (GRUB_BOOT_, GRUB_MACHINE, _KERNEL_SEG)

	#define GRUB_OFFSETS_CONCAT_(a,b,c) a ## b ## c
	#define GRUB_OFFSETS_CONCAT(a,b,c) GRUB_OFFSETS_CONCAT_(a,b,c)

	/* The segment where the kernel is loaded.  */
	#define GRUB_BOOT_I386_PC_KERNEL_SEG	0x800

同时(编译完代码后)，在 grub-core/Makefile 中有:

	TARGET_CPPFLAGS =  -Wall -W  -DGRUB_MACHINE_PCBIOS=1 -DGRUB_MACHINE=I386_PC -m32 bluhbluh...

这样就明白为什么 *GRUB_BOOT_MACHINE_KERNEL_ADDR* 的值是 0x8000 了。

BIOS 将控制权 transfer 到 grub 时会设置 DL 寄存器，指示一个 drive number，也即后续从哪个驱动器继续读取 grub kernel image。变量 boot_drive 的默认值是 0xff，根据注释可知，0xff 的意思是使用 (BIOS设置的)DL 中的值，但 grub-install 可以修改这个值。如果 boot_drive 的值不是 0xff，则 load boot_drive 到 DL。代码如下：

		/* Check if we have a forced disk reference here */
		movb   boot_drive, %al
		cmpb	$0xff, %al
		je	1f
		movb	%al, %dl
	1:
		/* save drive reference first thing! */
		pushw	%dx

driver number 属于 BIOS 的知识范畴，因为 bootloader 在使用 BIOS 的磁盘读写中断服务时才会使用这个值，所以 BIOS 对其有解释权，但没有找到准确的介绍，这几篇可以参考一下：

1. [list the BIOS drive index](https://stackoverflow.com/questions/45891044/any-way-to-list-the-bios-drive-numbers-in-real-mode)
2. [PC boot: dl register and drive number](https://stackoverflow.com/questions/11174399/pc-boot-dl-register-and-drive-number)
3. [BIOS to MBR interface](https://en.wikipedia.org/wiki/Master_boot_record#BIOS_to_MBR_interface)

简而言之，如果 boot drive 是硬盘，则 DL 的最高 bit 为 1，即驱动器号范围是 0x80 - 0x8f；如果是软盘，则最高 bit 为 0，即驱动器号范围是 0x0 - 0xF。

boot.S 第一指令是 jmp 到 after_BPB 处执行：

		jmp	LOCAL(after_BPB)
		LOCAL(after_BPB):

	/* general setup */
		cli		/* we're not safe here! */

	    /*
	     * This is a workaround for buggy BIOSes which don't pass boot
	     * drive correctly. If GRUB is installed into a HDD, check if
	     * DL is masked correctly. If not, assume that the BIOS passed
	     * a bogus value and set DL to 0x80, since this is the only
	     * possible boot drive. If GRUB is installed into a floppy,
	     * this does nothing (only jump).
	     */
	     /* 上面的注释说：若 grub 被安装在硬盘上，dl 唯一的有效值是 0x80，因为这是唯一可能
	      * 的 boot drive。如果是安装在软盘上的，这一段 do nothing. */
		.org GRUB_BOOT_MACHINE_DRIVE_CHECK
	boot_drive_check:
		/* 当 grub 安装在 HDD 时，某些有问题的 BIOS 会错误的设置 DL 寄存器，所以 jmp 指令
		 * 才可能被 overwrite，进入下一条指令进行相关判断。如果是安装在软盘时，则不存在这个
		 * 问题，直接跳到 3:, 进行软盘相关的判断。参考上面的文章 <BIOS to MBR interface> */
	    jmp     3f	/* grub-setup may overwrite this jump. */
	    testb   $0x80, %dl /* 如果 dl 的最高 bit 不是 1，结果为0，跳到 2，强赋值为 0x80 */
	    jz      2f
	3:
		/* Ignore %dl different from 0-0x0f and 0x80-0x8f.  */
		testb   $0x70, %dl /* 等分析 grub-bios-setup 后再来补充 */
		jz      1f
	2:
	    movb    $0x80, %dl
	1:
		/*
		 * ljmp to the next instruction because some bogus BIOSes
		 * jump to 07C0:0000 instead of 0000:7C00.
		 */
		ljmp	$0, $real_start

	real_start:
		xxx


真正的代码开始于 real_start：通过调用 INT 0x13 判断磁盘是否支持 LBA 访问模式，否则使用传统的 CHS 模式。以 LBA 为例，继续调用 INT 0x13，从 core.img 的第一个 sector 的起始位置读入一个 sector(代码注释是："the blocks") 到地址为 *GRUB_BOOT_MACHINE_BUFFER_SEG*(0x7000):0 的 buffer 中，关于这个 INT 0x13 调用的详细解释参考 [wikipedia](https://en.wikipedia.org/wiki/INT_13H) 或 [这篇](http://www.ctyme.com/intr/rb-0708.htm)。然后调用函数 *copy_buffer*，从源地址 *GRUB_BOOT_MACHINE_BUFFER_SEG*:0 拷贝 512 bytes 到 0:*GRUB_BOOT_MACHINE_KERNEL_ADDR*，也即从 0x7000:0 拷贝到 0:0x8000，然后 jmp 到 *GRUB_BOOT_MACHINE_KERNEL_ADDR*。

MBR 中的 boot.img 的工作就完成了。

#### BIOS INT 13h

顾名思义，BIOS 提供的 13h 号中断向量服务，主要功能是提供磁盘读写服务，参考[wikipedia](https://en.wikipedia.org/wiki/INT_13H)的介绍。boot.img 中使用了两次 INT 13h 服务，一是检查磁盘是否支持 LBA 读写模式，第二是读取磁盘上的 diskboot.img。简单了解一下细节。

第一处检查是否支持 LBA：

	Int 13/AH=41h/BX=55AAh: Check Extensions Present

它的作用是检查 **INT 13h Extensions** 功能是否存在，这个 Extension 是支持更大的磁盘空间访问，更大的磁盘空间访问是通过支持 LBA 模式实现。所以代码注释直接道出了本质：

	/* check if LBA is supported */

第二处从磁盘读取 diskboot.img：

	INT 13h AH=42h: Extended Read Sectors From Drive

使用指针 DS:SI 指向叫做 **disk address packet**(DAP) 的一块内存，INT 13h 的调用者需要初始化这段数据，INT 13h 从其中取得必要的入参进行操作，所以这块内存也被叫做 packet interface。参考[这里](https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH=42h:_Extended_Read_Sectors_From_Drive)详细了解 INT 13 和 DAP 的格式。

boot.s 开头的代码定义了 DAP:

	.macro scratch
	/* scratch space */
	mode:
		.byte	0
	disk_address_packet:
	sectors:
		.long	0
	heads:
		.long	0
	cylinders:
		.word	0
	sector_start:
		.byte	0
	head_start:
		.byte	0
	cylinder_start:
		.word	0
	/* more space... */
	.endm

可以看出格式并不遵循 DAP 定义，应该是还有其他用处，待分析。DAP 之前还有一个字段： mode，用于指示当前磁盘是否支持 LBA（1：支持，0：不支持），方便后面判断磁盘是否支持 LBA mode，因为在 boot.S 中已经通过 INT 13/AH=41h/BX=55AAh 判断过了，将结果保存在这个字段，后面的代码直接检查这个字段的值即可，而不用再使用 INT 13。

初始化 DAP 的代码如下：

	LOCAL(lba_mode):
		xorw	%ax, %ax
		movw	%ax, 4(%si)

		incw	%ax
		/* set the mode to non-zero */
		movb	%al, -1(%si)

		/* the blocks */
		movw	%ax, 2(%si)

		/* the size and the reserved byte */
		movw	$0x0010, (%si)

		/* the absolute address */
		movl	LOCAL(kernel_sector), %ebx
		movl	%ebx, 8(%si)
		movl	LOCAL(kernel_sector_high), %ebx
		movl	%ebx, 12(%si)

		/* the segment of buffer address */
		movw	$GRUB_BOOT_MACHINE_BUFFER_SEG, 6(%si)

一眼看上去感觉乱乱的，实际上应该是为了合理利用 ax 寄存器的值才使的顺序变这个样子。

### diskboot.img

由 core.img 的图示可知，它的第一个 sector 的内容是 diskboot.img。diskboot.img 对应的源代码文件是 grub-core/boot/i386/pc/diskboot.S。diskboot.img 的执行环境，也即寄存器，由 boot.img 设置，此时的环境如下：

1. 有可用的堆栈(SS 和 SP 已配置)。
2. 寄存器 DL 中保存正确的引导驱动器。
3. 寄存器 SI 保存着 DAP(Disk Address Packet) 的地址，因为还需要使用 INT 13 AH=42h 来继续读取磁盘 sector。

diskboot.img 的工作是将 core.img 中剩余的部分继续加载到内存，并跳转过去执行。diskboot.img 的工作本质上和 boot.img 一样，都是借助 BIOS 的 interrupt service 读取磁盘 sector 的内容到内存，只不过 diskboot.img 需要加载多个 sector 而已。

diskboot.img 需要知道 core.img 剩余部分所在的 sector，显然，这是安装 grub 的时候才会知道，grub-install 时将 core.img 占据的 sector 信息写入 diskboot.img，这部分空间定义在 diskboot.S 的尾部：

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

为什么这段空间被标以 label: firstlist 呢？一个 blocklist 描述一段连续的磁盘区域，而在某些情况下，core.img 有可能被分成多块安装在磁盘上，所以可能存在多个 blocklist，如果有多个的时候，这段空间会紧挨着 firstlist 向 diskboot.img 开始的方向延伸。 下面对代码做分析：


		/* this sets up for the first run through "bootloop" */
		/* 将 firstlist 地址保存到寄存器 di */
		movw	$LOCAL(firstlist), %di

		/* save the sector number of the second sector in %ebp */
		/* firstlist 处的第一个 long 数据，即第一个 sector 号，保存到 ebp.
		 * 还有4 byte 长的 sector 号呢？并且 diskboot.S 没有碰 ebp 寄存器*/
		movl	(%di), %ebp

	    /* this is the loop for reading the rest of the kernel in */
	    /* 因为每次读取的 sector 数有限制，所以需要循环读取 */
	LOCAL(bootloop):

		/* check the number of sectors to read */
		/* 8(%di)，即 firstlist + 8 处，保存的是待读取的 sector 数，
		 * 每次读取 sector 后，这里需要减去已读取的 sector 数。*/
		cmpw	$0, 8(%di)

		/* if zero, go to the start function */
		/* 如果 8(%di) 处的值是 0，说明待读取的 sector 为0，说明已经读取完，
		 * 那就可以跳转 bootit 启动了;否则继续读取 sector */
		je	LOCAL(bootit)

	LOCAL(setup_sectors):
		/* check if we use LBA or CHS */
		/* 继续读取 sector，判断驱动器是否支持 LBA 的快捷方式，在 boot.S 的分析中已做介绍 */
		cmpb	$0, -1(%si)

		/* use CHS if zero, LBA otherwise */
		je	LOCAL(chs_mode)

		/* load logical sector start */
		/* 将 core.img 剩余部分内容的起始地址(sector No. in LBA mode)放入寄存器 ebx
		 * 和 ecx，下面调用 INT 13 时候会用到 */
		movl	(%di), %ebx
		movl	4(%di), %ecx

		/* the maximum is limited to 0x7f because of Phoenix EDD */
		/* Phoenix EDD 的介绍可以参考 INT 13 的 wikipedia 页面：
		 *   To support even larger addressing modes, an interface known as
		 *   INT 13h Extensions was introduced by Western Digital and Phoenix
		 *   Technologies as part of BIOS Enhanced Disk Drive Services (EDD).
		 * 所以，由于它的限制，每次最多只能读取 0x7f 个 sector */
		xorl	%eax, %eax
		movb	$0x7f, %al

		/* how many do we really want to read? */
		/* 拿本次要读取的数目，和待读取的总数比较，即从 8(%di) 处减去要读取的数 */
		cmpw	%ax, 8(%di)	/* compare against total number of sectors */

		/* which is greater? */
		/* cmp 后结果不为0的话，说明就是要读取 0x7f 个 sector，则跳到 1f */
		jg	1f

		/* if less than, set to total */
		/* cmp 后发现结果 <= 0，说明待读取的数目小于 0x7f，那么就读取“待读取”的数目即可 */
		movw	8(%di), %ax

	1:
		/* subtract from total */
		subw	%ax, 8(%di)

		/* add into logical sector start */
		/* 寄存器 di 是 firstlist 的地址，地址的内容是待读取的 sector 的地址，也即 sector
		 * 号，每次读取 sector 后，下次待读取的地址自然要更新，新地址 = 老地址 + 上次读取的
		 * sector 数 */
		addl	%eax, (%di)
		adcl	$0, 4(%di)

		/* set up disk address packet */

		/* the size and the reserved byte */
		/* 初始化 DAP 区域，跟 boot.S 中一样，前两个字节是 0x10, 0，因为 x86 是小端，
		 * 所以使用立即数 0x0010 */
		movw	$0x0010, (%si)

		/* the number of sectors */
		movw	%ax, 2(%si)

		/* the absolute address */
		/* 填入 sector 读取地址，已经在上面代码初始化过 */
		movl	%ebx, 8(%si)
		movl	%ecx, 12(%si)

		/* the segment of buffer address */
		/* 还是使用和 boot.S 中一样的 buffer segment */
		movw	$GRUB_BOOT_MACHINE_BUFFER_SEG, 6(%si)

		/* save %ax from destruction! */
		pushw	%ax

		/* the offset of buffer address */
		/* 还是使用和 boot.S 中一样的 buffer segment 的 offset 0 */
		movw	$0, 4(%si)

		/* BIOS call "INT 0x13 Function 0x42" to read sectors from disk into memory */
		movb	$0x42, %ah
		int	$0x13

		jc	LOCAL(read_error)

		/* 读取 sector 到 memory 后，立刻调用 copy_buffer，把 buffer 中的内容
		 * cp 到它应该在的位置 */
		movw	$GRUB_BOOT_MACHINE_BUFFER_SEG, %bx
		jmp	LOCAL(copy_buffer)

略过 chs_mode 的代码，来到 copy_buffer:

	LOCAL(copy_buffer):
	/* 将刚刚读到 buffer 中的数据 cp 到目的地址 */
		/* load addresses for copy from disk buffer to destination */
		/* 10(%di) 处保存着 buffer 数据将被 cp 到目的段的段基址： GRUB_BOOT_MACHINE_KERNEL_SEG
		 * + 0x20，即 0x820。这里的策略是：每次 cp 使用的目的地址是 10(%di): 0，即每次
		 * offset 都是 0，那么每次 cp 后只更新段基址 10(%di) 即可形成下次 cp 的物理(线性)地址 */
		movw	10(%di), %es	/* load destination segment */

		/* restore %ax */
		/* ax 中保存着刚刚 INT 13 读取的磁盘 sector 数 */
		popw	%ax

		/* determine the next possible destination address (presuming 512 byte sectors!) */
		/* 为啥表示 sector 数的 %ax 左移 5 bit？每次从 buffer 中 cp %ax 个 sector 内容
		 * 到目的地址 10(%di):0，所以每次 cp 后，目的地址 = 目的地址 + %ax * 512，即 %ax
		 * 需要左移 9 bit。因为使用的策略是仅更新段基址，所以左移 5 bit 后的值加到 10(%di)
		 * 处的段基址上，又因为 logical address 转换 linear address 时，段基址还要左移
		 * 4 bit，所以其实一共左移 9 bit，也就达到了 %ax * 512(2^9) 的目的。 */
		shlw	$5, %ax		/* shift %ax five bits to the left */
		addw	%ax, 10(%di)	/* add the corrected value to the destination
							       address for next time */

		/* save addressing regs */
		pusha
		pushw	%ds

		/* get the copy length */
		/* 上面左移了 5 bit，这里又左移了 3 bit，共 8 bit，也即 %ax = %ax * 2^8 =
		 * %ax * 256。因为一个 sector 是 512 byte，且下面使用的是 movsw 指令，每次 mov
		 * 一个 word(2 bytes)，所以 cp 一个 sector 只需要 512 / 2 = 256(2^8) 次即可。
		 * 所以，一共移动 %ax << 8 次即可 */
		shlw	$3, %ax
		movw	%ax, %cx

		/* 将 DS：SI cp 到 ES:DI */
		xorw	%di, %di	/* zero offset of destination addresses */
		xorw	%si, %si	/* zero offset of source addresses */
		movw	%bx, %ds	/* restore the source segment */

		cld		/* sets the copy direction to forward */

		/* perform copy */
		rep		/* sets a repeat */
		movsw		/* this runs the actual copy */

		/* restore addressing regs and print a dot with correct DS
		   (MSG modifies SI, which is saved, and unused AX and BX) */
		popw	%ds
		MSG(notification_step)
		popa

		/* check if finished with this dataset */
		/* cp 完成后，检查还有没有待读取的 sector，有的话继续这个循环，跳回 setup_sectors */
		cmpw	$0, 8(%di)
		jne	LOCAL(setup_sectors)

		/* update position to load from */
		/* 如果检查发现这个 blocklist 中的 sector 都读取完了，那么查看是否还有 blocklist
		 * 需要读取，如果还有 blocklist，上面已经解释过，它将紧挨着 diskboot.S 文件底部的
		 * firstlist，所以减去 12 即可 */
		subw	$GRUB_BOOT_MACHINE_LIST_SIZE, %di

		/* jump to bootloop */
		/* 这时请在回到前文看 bootloop 处的注释 */
		jmp	LOCAL(bootloop)

	/* END OF MAIN LOOP */

一般情况下，只有一个 blocklist 数据结构，磁盘数据读取结束后，通过下面的代码跳转到地址 0:0x8200

	LOCAL(bootit):
	/* print a newline */
	MSG(notification_done)
	popw	%dx	/* this makes sure %dl is our "boot" drive */
	ljmp	$0, $(GRUB_BOOT_MACHINE_KERNEL_ADDR + 0x200)

跳转后的代码是 lzma_decompress.img 的内容。

### lzma_decompress.img

lzma_decompress.img 对应的源码是 grub-core/boot/i386/pc/startup_raw.S，此文件中又 include 同目录下的  "lzma_decode.S"，这是 lzma 的算法核心。它的工作是解压缩它后面的压缩代码，并跳转过去，由 core.img 的图示可知，跳转到 kernel.img，由名字可知，这是 grub 的核心代码，它对应的代码在 grub-core/kern 目录下。从某种意义上说，kernel.img 的代码才是 grub 真正的开始。对于 lzma_decompress.img 代码的详细分析参考[此文](https://blog.csdn.net/conansonic/article/details/78534950)。本节仅做简单分析。

startup_raw.S 的开头部分是一条跳转指令：

	ljmp $0, $ABS(LOCAL (codestart))

跳过开头部分的 special data area：*GRUB_DECOMPRESSOR_MACHINE_COMPRESSED_SIZE*， *GRUB_DECOMPRESSOR_MACHINE_UNCOMPRESSED_SIZE*，顾名思义，在 grub-mkimage 生成 core.img 时，由其将数据填写到此处。

	/* the real mode code continues... */
	LOCAL (codestart):
		cli		/* we're not safe here! */

		/* set up %ds, %ss, and %es */
		/* diskboot.S 最后的跳转指令中的 CS 值为 0，下面几条指令将 DS，SS，ES 也置 0*/
		xorw	%ax, %ax
		movw	%ax, %ds
		movw	%ax, %ss
		movw	%ax, %es

		/* set up the real mode/BIOS stack */
		/* boot.img 和 diskboot.img 中都使用 GRUB_BOOT_MACHINE_STACK_SEG(0x2000)
		 * 作为 sp 的值，这里为啥换了呢？ */
		movl	$GRUB_MEMORY_MACHINE_REAL_STACK, %ebp
		movl	%ebp, %esp

		sti		/* we're safe again */
		...

		/* transition to protected mode */
		/* 为了更大的内存访问空间，只能切换到 protect mode. */
		calll	real_to_prot
		...

		/* 下面为了调用C函数做准备，编译时使用了 mregparm = 3，所以使用寄存器以顺序
		 * EAX, EDX, ECX 来传递参数，对应C函数的入参。但这一步并不是解压缩，内容暂时不明。*/
		movl	LOCAL(compressed_size), %edx
	#ifdef __APPLE__
		addl    $decompressor_end, %edx
		subl    $(LOCAL(reed_solomon_part)), %edx
	#else
		addl    $(LOCAL(decompressor_end) - LOCAL(reed_solomon_part)), %edx
	#endif
		movl    reed_solomon_redundancy, %ecx
		leal    LOCAL(reed_solomon_part), %eax
		cld
		call    EXT_C (grub_reed_solomon_recover)
		jmp	post_reed_solomon

将紧挨着 lzma_decompress.img 的数据(开始于 decompressor_end)解压缩到临时解压缩区域 *GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR*(0x100000) 处，并跳转过去执行，代码如下：

	post_reed_solomon:

	#ifdef ENABLE_LZMA
		movl	$GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR, %edi /* edi：解压缩目的地址 */
	#ifdef __APPLE__
		movl	$decompressor_end, %esi
	#else
		movl	$LOCAL(decompressor_end), %esi /* esi：待解压的数据起始地址 */
	#endif
		pushl	%edi
		movl	LOCAL (uncompressed_size), %ecx /* ecx: 解压后的 size */
		leal	(%edi, %ecx), %ebx /* ebx: 解压后的数据最大不能超过的地址= 起始地址 + size */
	/* Don't remove this push: it's an argument.  */
		push 	%ecx
		call	_LzmaDecodeA
		pop	%ecx
		/* _LzmaDecodeA clears DF, so no need to run cld */
		popl	%esi /* 此时 esi 中是刚刚 push %edi 的值 */
	#endif

		movl	LOCAL(boot_dev), %edx
		movl	$prot_to_real, %edi
		movl	$real_to_prot, %ecx
		movl	$LOCAL(realidt), %eax
		jmp	*%esi

edi 被赋值为 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR(0x100000=1M)，然后被 push 到 stack 上保存，调用 _LzmaDcodeA 函数后又 `popl %esi`，最后 `jmp *%esi`，也即 jmp 到 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR。这个跳转指令使用了不常见的 `*%esi` 形式，语法解释在 9.15.7 of `info as`：AT&T absolute (as opposed to PC relative) jump/call operands are prefixed by '*'。这里又引入一个知识点：绝对跳转(absolute jump) vs 相对跳转(relative jump)，参考 intel 指令手册的 JMP 指令。

### kernel.img

看一下构建 kernel.img 的 Makefile 内容，在 grub-core/Makefile.core.am:

	# 为适配排版，格式有微调
	if COND_i386_pc
	platform_PROGRAMS += kernel.exec
	kernel_exec_SOURCES  = kern/i386/pc/startup.S
	kernel_exec_SOURCES += kern/i386/pc/init.c kern/i386/pc/mmap.c term/i386/pc/console.c
	               kern/i386/dl.c kern/i386/tsc.c kern/i386/tsc_pit.c kern/compiler-rt.c
	               kern/mm.c kern/time.c kern/generic/millisleep.c kern/command.c
	               kern/corecmd.c kern/device.c kern/disk.c kern/dl.c kern/env.c
	               kern/err.c kern/file.c kern/fs.c kern/list.c kern/main.c kern/misc.c
	               kern/parser.c kern/partition.c kern/rescue_parser.c kern/rescue_reader.c
	               kern/term.c
	...
	kernel_exec_LDFLAGS  = $(AM_LDFLAGS) $(LDFLAGS_KERNEL) $(TARGET_IMG_LDFLAGS)
	                       $(TARGET_IMG_BASE_LDOPT),0x9000
	...

可以看出：kernel.img 的入口是 startup.S，其代码起始(虚拟)地址为 0x9000，因为 kernel.img 运行在保护模式下，所以文件开头有 directive `.code32`。

在 kernel.img 之前运行的代码的 CS 的值都是0，切换到保护模式后，CS 的 segment descriptor 基址也是0。由 startup_raw.S 最后几行代码可知，此时 esi 的值是 0x100000(1M)，edi，ecx 保存着两个函数的地址，eax 保存数据 realidt 的地址。在开头 startup.S 的开头，这几个地址被保存到 kernel.img 内部的变量中：

		movl	%ecx, (LOCAL(real_to_prot_addr) - _start) (%esi)
		movl	%edi, (LOCAL(prot_to_real_addr) - _start) (%esi)
		movl	%eax, (EXT_C(grub_realidt) - _start) (%esi)

然后 copy 自己回到链接指定的地址 0x9000 处：

		/* copy back the decompressed part (except the modules) */
	#ifdef __APPLE__
		movl	$EXT_C(_edata), %ecx
		subl    $LOCAL(start), %ecx
	#else
		/* 二者相减，得到 kernel.img 的 size，存在 ecx，下面的 movsb 指令会用到 */
		movl	$(_edata - _start), %ecx
	#endif
		/* _start 位于 kernel.img 的开头，所以它的地址是 0x9000，这是 copy 的目的地址 */
		movl	$(_start), %edi
		rep
		movsb

		/* 窍门：符号 cont 的地址是基于 0x9000，加上当前的 location counter。刚刚已经把
		 * kernel.img copy 回它应该在的地址，这里通过绝对跳转，跳回符号 cont 的地址继续执行。
		 * 也就是说，绝对跳转后执行的代码是取自 0x9000 的 kernel.img，不是 0x100000(buffer中)
		 * kernel.img */
		movl	$LOCAL (cont), %esi
		jmp	*%esi
	LOCAL(cont):

这里有个不常见的符号: _edata，可以通过 `man etext/edata/end/` 来了解，ld 的默认链接脚本会定义了这几个符号。那么如何查看 ld 的 default linker script 呢？ 使用 `ld --verbose` 可以查看默认链接脚本的完整内容。继续看后面的代码

	LOCAL(cont):
	...

		/* clean out the bss */
		/* 跳转回来后第一件事是为 bss section 手动清零，BSS_START_SYMBOL 的值默认是
		 * __bss_start，END_SYMBOL 默认值是 "end"，都在 ld 的默认链接脚本中定义。*/
		movl	$BSS_START_SYMBOL, %edi

		/* compute the bss length */
		movl	$END_SYMBOL, %ecx

		/* 相减得到 size，存到 ecx，下面的 rep 会用到 */
		subl	%edi, %ecx

		/* clean out */
		xorl	%eax, %eax
		cld
		rep
		stosb /* store eax to ES:EDI，每次 1 byte，重复 ecx 次*/

		movl	%edx, EXT_C(grub_boot_device)

		/*
		 *  Call the start of main body of C code.
		 */
		call EXT_C(grub_main)

移动完 kernel.img，清零 bss 区域，然后跳到 C 函数 *grum_main*，这才进入 grub 的核心内容。分析到这一步，可以暂停下，先去阅读“安装 GRUB”一节，再回来看 kernel.img 的流程。

#### grub_main

从这个函数开始是 grub 核心内容

	void __attribute__ ((noreturn)) grub_main (void)

函数定义可知，它是不会返回的。下面依然分析捡重点代码分析。grub_main() 的主要过程如下面的代码所示

	void __attribute__ ((noreturn)) grub_main (void)
	{
	  /* First of all, initialize the machine.  */
	  grub_machine_init ();
	  ...
	  /* 若 module info data 中有 OBJ_TYPE_CONFIG 类型(一般没有)的对象，此函数才有用 */
	  grub_load_config ();
	  ...
	  /* 函数顾名思义，kernel.img 中有专门的数据结构 grub_symtab 来保存一些符号，目前看来
	   * 在 module 重定位时会用到。此函数将 kernel.img 自身的一些符号信息先注册到 grub_symtab
	   * 数据结构中。待确认和详细分析 */
	  grub_register_exported_symbols();
	  ...
	  /* 此函数将所有 buffer 中的 module 加载并链接起来，使其可用。内容复杂，下面有单独分析 */
	  grub_load_modules ();
	  ...
	  /* Reclaim space used for modules.
	   * buffer 中的 module 使用完了，buffer 空间就没用了，回收 */
	  reclaim_module_space ();
	  ...
	  /* 注册一些核心命令，这些命令是由 kernel.img 自己提供。其他命令还是由各 module 提供 */
	  grub_register_core_commands ();
	  ...
	  /* 加载 grub 目录中的 normal 模块，其加载过程和函数 grub_load_modules 中一样。加载
	   * 后则执行 normal 命令。normal 模块会加载 linux kernel, initramfs */
	  grub_load_normal_mode ();
	}

第一个函数 grub_machine_init 处理的事情比较多，拿出来单独分析：

	void grub_machine_init (void)
	{
	  /* 针对 VIA 的芯片做 workaround，无需研究 */
	  grub_via_workaround_init ();

	  /* 上文已说，解压缩的临时 buffer 位于地址 1M(0x100000)处，压缩数据前端是 kernel.img，
	   * 后面是各 module。bss section 不占据文件空间，所以 _edata - _start 是 kernel.img
	   * 的有效 size；又因 module 紧挨 kernel.img，所以 modules 的起始地址计算如下 */
	  grub_modbase = GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR + (_edata - _start);
	  ...

	  /* 内存的初始化是重点。其实比较简单，函数 grub_machine_mmap_iterate 通过 e820 中断
	   * 获得内存信息(addr, len, type)，然后将信息交给函数 mmap_iterate_hook 处理，过滤
	   * 掉地址小于 1M 且大于 4G 的，将类型是 GRUB_MEMORY_AVAILABLE 的内存区域保存到数组
	   * mem_regions[] 中 */
	  grub_machine_mmap_iterate (mmap_iterate_hook, NULL);
	  /* 整理数组 mem_regions[]：按地址从小到大排序，如果 2 块内存区域有重叠，则合二为一 */
	  compact_mem_regions ();

	  /* 获得解压缩 buffer 中的 module 的结束地址，通过 grub_mm_init_region 初始化 grub
	   * 的内存管理功能。要看明白此函数，需要了解 core.img 的打包格式，在“安装 GRUB”一节中有
	   * 图示。因为不是重点，所以对此函数不做详细。看起来由两级数据结构来管理：grub_mm_region_t
	   * & grub_mm_header_t。初始化后，grub 中所有的 malloc 类操作都是在操作 grub_mm_base
	   * 这个数据结构 */
	  modend = grub_modules_get_end ();
	  for (i = 0; i < num_regions; i++)
	  {
		grub_addr_t beg = mem_regions[i].addr;
		grub_addr_t fin = mem_regions[i].addr + mem_regions[i].size;
		if (modend && beg < modend)
		beg = modend;
	    if (beg >= fin)
		continue;
	    grub_mm_init_region ((void *) beg, fin - beg);
	  }
	}

grub_load_modules 函数比较复杂，核心内容是遍历 buffer 中所有类型为 OBJ_TYPE_ELF 的 module，将其代码和数据加载到一块分配的内存中并进行重定位，然后执行 module 的初始化函数。

>grub_load_modules -- 遍历module --> grub_dl_load_core --> **grub_dl_load_core_noinit**
                                                     \
>                                                     --> grub_dl_init

重点在函数 grub_dl_load_core_noinit 中:

	/* 入参 addr 是 module 在 buffer 中的地址 */
	grub_dl_t grub_dl_load_core_noinit (void *addr, grub_size_t size)
	{
	  Elf_Ehdr *e;/* module 是 ELF 文件 */
	  grub_dl_t mod;/* 用来描述一个 module 的总结构体 */
	  ...
	  /* ELF header 的常规检查，很常规，不做分析 */
	  grub_dl_check_header(e, size);
	  ...

	  /* 重要的处理在下面这些函数中，总的来说是对 ELF 文件的 section 做操作。要看懂下面的函数，
	   * 需要仔细阅读 `man elf`。grub_dl_check_license 和 grub_dl_resolve_name 的内容
	   * 很简单，略过；grub_dl_resolve_dependencies 也比较简单，找一下是否有 module 依赖
	   * 关系的 section，如果有则先加载依赖的 module；其余三个函数工作量比较大，重点分析。*/
	  if (grub_dl_check_license (e)
		  || grub_dl_resolve_name (mod, e)
		  || grub_dl_resolve_dependencies (mod, e)
		  || grub_dl_load_segments (mod, e)
		  || grub_dl_resolve_symbols (mod, e)
		  || grub_dl_relocate_symbols (mod, e))
	  {
		mod->fini = 0;
		grub_dl_unload (mod);
		return 0;
	  }
	}

	/* 将所有包含代码和数据的 section 从 buffer 加载到分配的内存 */
	static grub_err_t grub_dl_load_segments (grub_dl_t mod, const Elf_Ehdr *e)
	{
	  ...
	  /* 遍历所有 section 获得 total size 和 total align，total align 是所有 alignment
	   * 中最大的那个，total size 是将所有 section size 对其到 alignment 后的和 */
	  for (i = 0, s = (const Elf_Shdr *)((const char *) e + e->e_shoff);
		   i < e->e_shnum;
	  	   i++, s = (const Elf_Shdr *)((const char *) s + e->e_shentsize))
	  {
	    tsize = ALIGN_UP (tsize, s->sh_addralign) + s->sh_size;
	    if (talign < s->sh_addralign)
		talign = s->sh_addralign;
	  }
	  /* 按 total size 和 total align 分配内存 */
	  mod->base = grub_memalign (talign, tsize);
	  mod->sz = tsize;
	  ptr = mod->base;

	  /* 再次遍历 module 的所有 section */
	  for (i = 0, s = (Elf_Shdr *)((char *) e + e->e_shoff);
		   i < e->e_shnum;
		   i++, s = (Elf_Shdr *)((char *) s + e->e_shentsize))
	  {
	    /* SHF_ALLOC 意思是这个 section occupies memory during process execution */
		if (s->sh_flags & SHF_ALLOC)
		{
		  grub_dl_segment_t seg; /* 被加载的 sections 用这个结构体来描述 */

		  seg = (grub_dl_segment_t) grub_malloc (sizeof (*seg));
		  if (! seg)
		    return grub_errno;

		  if (s->sh_size)
		  {
	        void *addr;
			/* 将目的地址按对齐需求对齐 */
	        ptr = (char *) ALIGN_UP ((grub_addr_t) ptr, s->sh_addralign);
	        addr = ptr;
	        ptr += s->sh_size;

			/* 然后 copy 到目的内存中 */
	        switch (s->sh_type)
			{
			case SHT_PROGBITS:
			  grub_memcpy (addr, (char *) e + s->sh_offset, s->sh_size);
			  break;
			case SHT_NOBITS:
			  grub_memset (addr, 0, s->sh_size);
			  break;
			}

	        seg->addr = addr;
	      }
		  else
		    seg->addr = 0;

		  seg->size = s->sh_size;
		  seg->section = i;
		  seg->next = mod->segment;
		  mod->segment = seg;
		}
	  }
	}

	/* 此函数对 buffer 中的 symbol table section 解析并原地(in place)修改 */
	static grub_err_t grub_dl_resolve_symbols (grub_dl_t mod, Elf_Ehdr *e)
	{
	  /* 遍历 sections 找到符号表 section */
	  for (i = 0, s = (Elf_Shdr *) ((char *) e + e->e_shoff);
	       i < e->e_shnum;
	   	   i++, s = (Elf_Shdr *) ((char *) s + e->e_shentsize))
		if (s->sh_type == SHT_SYMTAB)
		  break;

	  /* 记下符号表在 buffer 中的地址 */
	  mod->symtab = (Elf_Sym *) ((char *) e + s->sh_offset);
	  /* 记下符号表 entry 的 size */
	  mod->symsize = s->sh_entsize;
	  ...
	  /* sh_link 在 man elf 并没有详细解释，详细解释参考：
	  * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html#chapter6-47976
	  * 对于符号表 section 来说，sh_link 是其对应 string table 的 section index */
	  s = (Elf_Shdr *) ((char *) e + e->e_shoff + e->e_shentsize * s->sh_link);
	  str = (char *) e + s->sh_offset; /* 拿到 string table 的地址 */

	  /* 遍历符号表中的每一个 entry，进行解析 */
	  for (i = 0;
	       i < size / entsize;
	       i++, sym = (Elf_Sym *) ((char *) sym + entsize))
	  {
	    switch (type)
	    {
		  case STT_NOTYPE:
		  case STT_OBJECT:

		  /* Resolve a global symbol.  */
		  /* https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblh/index.html,
		   * table 7-11. 如果符号有名字，且 st_shndx 是 SHN_UNDEF(0)，说明是定义在别处
		   * (module or kernel.img)的符号 */
		  if (sym->st_name != 0 && sym->st_shndx == 0)
		  {
			/* 未在本 module 中定义的 symbol 须已注册在内部的 grub_symtable 中，否则错误返回 */
			grub_symbol_t nsym = grub_dl_resolve_symbol (name);
	        if (! nsym)
			  return grub_error (GRUB_ERR_BAD_MODULE,
					   N_("symbol `%s' not found"), name);

			  /* 找到后则把符号地址赋值给 st_value；并更新 symbol entry 的 st_info field */
		      sym->st_value = (Elf_Addr) nsym->addr;
		      if (nsym->isfunc)
				sym->st_info = ELF_ST_INFO (bind, STT_FUNC);
		  }
		  else
		  {
		    /* 此 symbol 在本 module 中定义 */
		    /* 关于 symbol table entry 中的 st_value 的定义，参考：
		     * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html#chapter6-35166
		     * 本例中，st_value 是符号在所在 section 中的 offset，加上 section address，
		     * 构成 st_value */
		    sym->st_value += (Elf_Addr) grub_dl_get_section_addr (mod,
								    sym->st_shndx);
			/* 若不是 local 的符号，则注册到 grub_symtable 中 */
		    if (bind != STB_LOCAL)
			  if (grub_dl_register_symbol (name, (void *) sym->st_value, 0, mod))
				return grub_errno;
		  }

		case STT_FUNC:
	    sym->st_value += (Elf_Addr) grub_dl_get_section_addr (mod, sym->st_shndx);

	    case...
		}// switch()
	  }// for()
	}

	/* 遍历 buffer 中 module 的重定位 section，对 cp 到已分配内存中代码/数据进行重定位 */
	static grub_err_t grub_dl_relocate_symbols (grub_dl_t mod, void *ehdr)
	{
	  for (i = 0, s = (Elf_Shdr *) ((char *) e + e->e_shoff);
	 	   i < e->e_shnum;
	   	   i++, s = (Elf_Shdr *) ((char *) s + e->e_shentsize))
		if (s->sh_type == SHT_REL || s->sh_type == SHT_RELA)
	    {/* 遍历找到 SHT_REL & SHT_RELA 的 section */
		  grub_dl_segment_t seg;
		  grub_err_t err;

		  /* Find the target segment.  */
		  /* 对于 sh_info 的详细解释，参考：
		   * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html#chapter6-47976
		   * 对于类型是 SHT_REL/SHT_RELA 的 section 来说，sh_info 意思是：
		   * The section header index of the section to which the relocation applies.*/
		  for (seg = mod->segment; seg; seg = seg->next)
		    if (seg->section == s->sh_info)
		      break;

		  /* 找到了要需要重定位的 section，调用函数进行重定位。注意，这个 section 的地址是
		   * 之前 grub_dl_load_segments 后分配的，而不是解压缩 buffer 中的。*/
		  if (seg)
		  {
	        if (!mod->symtab)
	          return grub_error (GRUB_ERR_BAD_MODULE, "relocation without symbol table");

		      err = grub_arch_dl_relocate_symbols (mod, ehdr, s, seg);
		      if (err)
		        return err;
	      }
	    }
	}

	/* 重定位函数，不同 ABI 的实现不同。x86 下分 i386 和 x86_64，以 i386 为例进行分析 */
	grub_err_t grub_arch_dl_relocate_symbols (grub_dl_t mod, void *ehdr, Elf_Shdr *s,
	                                          grub_dl_segment_t seg)
	{
	  Elf_Rel *rel, *max;

	  /* 遍历每一个 relocation entry */
	  for (rel = (Elf_Rel *) ((char *) ehdr + s->sh_offset),
				 max = (Elf_Rel *) ((char *) rel + s->sh_size);
		   rel < max;
		   rel = (Elf_Rel *) ((char *) rel + s->sh_entsize))
	  {
	    Elf_Word *addr;
	    Elf_Sym *sym;

	    if (seg->size < rel->r_offset)
		  return grub_error (GRUB_ERR_BAD_MODULE,
			   "reloc offset is out of the segment");

		/* 找到重定位点的起始地址: section 地址 + 重定位点在 section 中的 offset。
		 * 拿到重定位点相关的符号信息 */
	    addr = (Elf_Word *) ((char *) seg->addr + rel->r_offset);
	    sym = (Elf_Sym *) ((char *) mod->symtab
			  + mod->symsize * ELF_R_SYM (rel->r_info));

		/* 根据重定位类型进行重定位。在之前的函数 grub_dl_resolve_symbols 中，已将 buffer
		 * 中符号表 entry 的 st_value 修改为符号的地址。下面的重定位也很容易理解，不赘述 */
	    switch (ELF_R_TYPE (rel->r_info))
		{
		case R_386_32:
		  *addr += sym->st_value;
		  break;

		case R_386_PC32:
		  *addr += (sym->st_value - (grub_addr_t) addr);
		  break;
		default:
		  return grub_error (GRUB_ERR_NOT_IMPLEMENTED_YET,
			     N_("relocation 0x%x is not implemented yet"),
			     ELF_R_TYPE (rel->r_info));
		}
	  }
	}

加载完 module 则执行其初始化函数：

	static inline void grub_dl_init (grub_dl_t mod)
	{
	  if (mod->init)
		(mod->init) (mod);

	  mod->next = grub_dl_head;
	  grub_dl_head = mod;
	}

Ok，终于介绍完了 grub_load_modules 函数，至此，我们可以对上述过程用下图做一个总结：

![grub boot process](grub_boot.png)

回头继续看 grub_main。倒数第二个函数是 grub_load_normal_mode，即加载 normal 模块并执行 normal 命令，关于 grub module 可以参考阅读"GRUB modules 简介"一节。normal module/command 的介绍参考[官方文档](https://www.gnu.org/software/grub/manual/grub/grub.html#normal)。

作为 kernel.img/grub_main 过程重要的一部分，我们仍然捡重点代码分析 normal 命令的过程：

	static grub_err_t grub_cmd_normal (struct grub_command *cmd __attribute__ ((unused)),
	                                   int argc, char *argv[])
	{
	  /* 由 grub_load_normal_mode 函数可知，本函数的入参 argc 和 argv 都是 0。
	   * 本函数的作用主要是读取 grub 目录中的 grub.cfg */
	  char *config;

	  config = grub_xasprintf ("%s/grub.cfg", prefix);
	  grub_enter_normal_mode (config);
	}

	/* This starts the normal mode.  */
	void grub_enter_normal_mode (const char *config)
	{
	  ...
	  /* 读取 grub 目录中各种文件。读取 command.lst，fs.lst，crypto.lst，terminal.lst
	   * 保存到内部数据结构中；读取 grub.cfg 保存到 grub_menu_t 结构中，并执行其中的命令
	   * (应该是 grub.cfg 上方，menuentry 以外的那些命令)，grub_menu_t 结构包括了显示
	   * grub menu 所需的数据。显示 menu 菜单，获得用户选择的 menu entry 或 timeout 后的
	   * default entry，执行这个 entry 中的各命令来启动 OS */
	  grub_normal_execute (config, 0, 0);
	  /* 正常情况下，下面这个函数不会走到？因为上面的函数已经成功启动 OS 了，只有在无法启动 OS
	   * 的异常情况下，grub_normal_execute 才返回？ */
	  grub_cmdline_run (0, 1);
	  ...
	}

展示 grub menu，获得 menu entry 并执行这个 entry 的 callchain 长这样(待修改为图片)：

>grub_normal_execute
    --> grub_show_menu
        --> show_menu
            --> boot_entry = run_menu (menu, nested, &auto_boot);
            --> e = grub_menu_get_entry (menu, boot_entry);
            --> grub_menu_execute_with_fallback (menu, e, autobooted, &execution_callback,0);
            --> grub_menu_execute_entry /* Run a menu entry */
                --> grub_script_execute_new_scope
                    /* 看起来像在逐行解析 menu entry 的内容，并执行对应的命令，
                     * 如 linux16, initrd16 等 */
                    --> grub_script_execute_sourcecode
                /* 执行完 entry 中的各种命令，可以启动 OS 了 */
                --> grub_command_execute ("boot", 0, 0)

上述过程有待详细分析。分析到这一步，我们所关心的 grub 工作流程，就剩下 menu entry 中用于加载 linux kernel 和 initramfs 的两条命令比较重要，将单独作为一节进行分析，因为它涉及 Linux kernel 的内容，将在 “normal 模块加载 linux kernel & initramfs”一节中进行分析。

#### GRUB modules introduction

Module 的概念在 grub2 中引入，有两篇文章可以作为科普：

1. [Writing GRUB Modules](https://wiki.osdev.org/Writing_GRUB_Modules)
2. [grub2-modules](http://blog.fpmurphy.com/2010/06/grub2-modules.html?output=pdf)

我们从代码的角度简单分析 module 的实现框架。每个 module 都需要 initialization 和 finalization 函数，分别由宏 GRUB_MOD_INIT 和 GRUB_MOD_FINI 辅助完成，他们的定义在 include/grub/dl.h 中：

	/* 为了简洁，直接列出在 i386-pc 平台上该宏的定义 */
	#define GRUB_MOD_INIT(name)	\
	static void grub_mod_init (grub_dl_t mod __attribute__ ((unused))) __attribute__ ((used)); \
	void \
	grub_##name##_init (void) { grub_mod_init (0); } \
	static void \
	grub_mod_init (grub_dl_t mod __attribute__ ((unused)))

	#define GRUB_MOD_FINI(name)	\
	static void grub_mod_fini (void) __attribute__ ((used)); \
	void \
	grub_##name##_fini (void) { grub_mod_fini (); } \
	static void \
	grub_mod_fini (void)

可以看出，对于 initialization 和 finalization，每个 module 都定义了 static 的函数: grub_mod_init() 和 grub_mod_fini。在上文加载模块的 grub_dl_resolve_symbols 函数中有如下这么一段：

	if (grub_strcmp (name, "grub_mod_init") == 0)
	  mod->init = (void (*) (grub_dl_t)) sym->st_value;
	else if (grub_strcmp (name, "grub_mod_fini") == 0)
	  mod->fini = (void (*) (void)) sym->st_value;

所以现在应该可以理解 grub_dl_init 函数了。以 normal 模块为例(grub-core/normal/main.c)，它的 initialization 函数内容很简单，基本都在注册命令，比如我们最关心的一个命令：

	/* Register a command "normal" for the rescue mode.  */
	grub_register_command ("normal", grub_cmd_normal, 0, N_("Enter normal mode."));

grub_main 函数的最后一步就是执行这个命令。

### "normal" mod loading linux kernel & initramfs

grub-mkconfig 生成 grub.cfg 时，会根据实际环境在 menuentry 中使用 linux16/initrd16 或者 linux/initrd 命令，究竟如何决定，代码细节尚未分析，也暂时略过。现在只需要知道，他们分别对应了 16-bit/32-bit 的 linux/x86 boot protocal 即可，boot protocol 在 linux kernel 的 Documentation/x86/boot.txt 中有详细介绍。本文将以 16-bit boot protocol 为例进行代码分析。

上文已经说过，normal 模块解析 grub.cfg 的 menu entry，然后执行选中的 entry，即执行 entry 中的命令。其中 linux 和 initrd 命令是用来加载 linux kernel 和 initramfs，本文以 linux16/initrd16 为例进行代码分析。在 grub.cfg 的 menu entry 中，这两条命令一般长这样：

>linux16 /vmlinuz-4.15.16-300.fc27.x86_64 root=UUID=ad9010cd-848d-4473-94c8-129d6e4a8cfe ro rhgb quiet

>initrd16 /initramfs-4.15.16-300.fc27.x86_64.img

linux16 命令由 1inux16 模块提供，代码在 grub-core/loader/i386/pc/linux.c 中：

	static grub_err_t grub_cmd_linux (grub_command_t cmd __attribute__ ((unused)), int argc, char *argv[])
	{
	  struct linux_i386_kernel_header lh;
	  ...
	  /* argv[0] 是 linux16 命令后的第一个参数，即要加载的 bzImage 文件。打开，获得文件 Handle */
	  file = grub_file_open (argv[0]);
	  ...
	  /* 然后读取 bzImage 头部的数据，存放在 linux_i386_kernel_header 结构中，这个结构
	   * 包含 boot protocol 的内容，在 linux kernel 文档中有详细描述，不是本节的分析重点。
	   * 读取后对部分数据做判断 */
	  if (grub_file_read (file, &lh, sizeof (lh)) != sizeof (lh))
	  ...

	  /* 初始化部分变量 */
	  ...
	  /* Documentation/x86/boot.txt: protocol 2.05 and earlier, 最大值是 255。
	   * 这里 256 是包括了结尾的 0 */
	  maximal_cmdline_size = 256;

	  /* protocol 2.00 是个分界线，之前不支持 bzImage & initrd */
	  if (lh.header == grub_cpu_to_le32_compile_time (GRUB_LINUX_I386_MAGIC_SIGNATURE)
		  && grub_le_to_cpu16 (lh.version) >= 0x0200)
	  {
	    /* 给 bzImage 中的 real mode 部分(setup.bin)分配内存空间，后面会详细分析。此时
		 * 只需知道将找一块结束地址 < 0xa0000 且 size >= GRUB_LINUX_CL_OFFSET +
		 * maximal_cmdline_size 的区域，用于加载 real mode 部分。为什么是 0xa0000？参考
		 * Documentation/x86/boot.txt 中 bzImage memory layout 的图示。这里有一点需要
		 * 注意，同样是表示 real mode 部分的地址，变量 grub_linux_real_target 容易和下面的
		 * grub_linux_real_chunk 混淆，仔细看会发现前者的类型是整数，典型的用作跳转指令的
		 * 操作数；后者是指针 char *，用于内存读写操作。所以下面会做转换。*/
		grub_linux_real_target = grub_find_real_target ();
		....
	  }

	  /* 使用了神秘的 relocator 机制，将整数类型的 grub_linux_real_target 转换为指针类型
	   * 的 grub_linux_real_chunk。前者只是根据 E820 的信息找出一块符合要求(大小，类型是可用)
	   * 的内存，但是这很粗糙。经过神秘的 relocator 机制，与 grub 的内存管理机制做了一次交互，
	   * 也就是说分配内存的事情还是要通过 grub 的内存管理机制。待明确 */
	  relocator = grub_relocator_new ();
	  if (!relocator)
		goto fail;

	  grub_relocator_chunk_t ch;
	  err = grub_relocator_alloc_chunk_addr (relocator, &ch,
		  			     grub_linux_real_target,
					     GRUB_LINUX_CL_OFFSET + maximal_cmdline_size);

	  ...
	  grub_linux_real_chunk = get_virtual_current_address (ch);

	  /* Put the real mode code at the temporary address. */
	   * 如官方注释所说，将 linux kernel 的 real mode 代码加载到内存中，先把
	   * linux_i386_kernel_header 的数据放入，然后将剩余部分从文件读出放入内存 */
	  grub_memmove (grub_linux_real_chunk, &lh, sizeof (lh));
	  len = real_size + GRUB_DISK_SECTOR_SIZE - sizeof (lh);
	  if (grub_file_read (file, grub_linux_real_chunk + sizeof (lh), len) != len)
	  {
	    ...
	  }
	  ...

	  /* Create kernel command line. */
	  /* 在 real mode 部分(setup.bin)空间后面写入 linux kernel 的 command line */
	  grub_memcpy ((char *)grub_linux_real_chunk + GRUB_LINUX_CL_OFFSET,
			LINUX_IMAGE, sizeof (LINUX_IMAGE));
	  /* 这里的 -1 是因为 LINUX_IMAGE 表示的字符串末尾是空字符(0)结尾 */
	  grub_create_loader_cmdline (argc, argv,
			      (char *)grub_linux_real_chunk
			      + GRUB_LINUX_CL_OFFSET + sizeof (LINUX_IMAGE) - 1,
			      maximal_cmdline_size
			      - (sizeof (LINUX_IMAGE) - 1));

	  /* 将 bzImage 的 protect mode 部分加载到地址： GRUB_LINUX_BZIMAGE_ADDR(0x100000)
	   * 或 GRUB_LINUX_ZIMAGE_ADDR(0x10000)。省略代码分析*/
	  ...

	  if (grub_errno == GRUB_ERR_NONE)
	  {
	    /* 最后一步，注册启动 OS 的 callback 函数: grub_linux16_boot，并 set 各种标记:
	     * loaded, grub_loader_flags, grub_loader_loaded 后续使用 */
		grub_loader_set (grub_linux16_boot, grub_linux_unload, 0);
		loaded = 1;
	  }
	}

	/* 给 bzImage 的 real mode 部分找一个合适的加载地址 */
	static grub_addr_t grub_find_real_target (void)
	{
	  grub_uint64_t result = (grub_uint64_t) -1;

	  /* 因为考虑了很多不同的情况，此函数比较复杂。我们以最简单的情况分析，其过程可以这样理解：
	   * 通过 E820 获得 memory map 的信息，交给 target_hook 函数处理选择：
	   *   1. 类型是 GRUB_MEMORY_AVAILABLE
	   *   2. 结束地址小于 0xa0000
	   *   3. size 必须大于 GRUB_LINUX_CL_OFFSET + maximal_cmdline_size，若某块 memory
	   *      size 大于它，则从这块 memory 尾端开始留出这个 size 的区域，目的如原注释所说：
	   *      Put the real mode part at as a high location as possible
	   * 按照这个要求遍历处理 E820 得到的所有 ENTRY，得到 the highest location。
	   */
	  grub_mmap_iterate (target_hook, &result);
	  return result;
	}

有一个细节需要知道：对于 linux kernel 的 real mode 部分，即 setup.bin 的 size，其实包括了两部分，一是开头的 512 bytes，由于历史的原因，被称为 boot sector，因为最早的 linux kernel 自带 boot sector，可以直接由 bios 启动；二是剩余部分，被称为 setup 代码，这部分的 size 由 boot protocol 中的 setup_sects 指示，单位如它的名字所示，是 sector。

从 grub 的代码来看，bzImage 的 real mode 部分最大是 GRUB_LINUX_MAX_SETUP_SECTS(64) x GRUB_DISK_SECTOR_BITS(512) = 32k，包括开头 512 bytes 的 boot sector，所以 setup code 实际最大只有 31k；从 linux 的文档 [Documentation/x86/boot.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt) 中的 bzImage memory layout 也可以看出 linux kernel 的 setup + boot sector 的大小是 0x8000，即 32k；更确凿的证据在 arch/x86/boot/setup.ld 中：

	. = ASSERT(_end <= 0x8000, "Setup too big!");

代码实际是按照 GRUB_LINUX_CL_OFFSET(0x9000) + maximal_cmdline_size = 36k + maximal_cmdline_size 来分配内存的，多出来的 (36k - 32k = 4k) 是留作 stack & heap 用，但他们的界限要等到进入 linux kernel 后才能确定(下方章节的 init_heap 函数)。grub_cmd_linux 函数中有：

	lh.heap_end_ptr = grub_cpu_to_le16_compile_time (GRUB_LINUX_HEAP_END_OFFSET);

即 heap_end_ptr = 0x9000 - 0x200 = 36k - 0x200(512)，linux boot protocol 对 heap_end_ptr 的解释是：

>Set this field to the offset (from the beginning of the real-mode code) of the end of the setup stack/heap, minus 0x0200.

不知道为什么要减去 0x200。在启动 OS 的函数 *grub_linux16_boot* 中有：

	state.sp = GRUB_LINUX_SETUP_STACK;

即 sp = 0x9000，因为 stack 的使用是从高地址 -> 低地址，所以把 sp 赋值为 36k 空间的最顶端。

initrd16 命令对应的函数 grub_cmd_initrd 相对简单一些，从起始于 GRUB_LINUX_BZIMAGE_ADDR + grub_linux16_prot_size，结束于 GRUB_LINUX_INITRD_MAX_ADDRESS - 0x10000(最简单情况下) 的大区域中选择一块尽量靠近尾部的区域，把 initrd image 加载进来。然后将 initrd image 加载的地址和 size 填写到 linux_i386_kernel_header 结构中相应的字段。

linux kernel 和 initrd image 都加载好了，现在可以回到 normal 模块继续看。上文讲到，normal 模块会解析 grub.cfg 中的 OS menu entry，并执行选中的 entry 中所有的命令(包括 linux16/initrd16)后，启动 OS。代码如下：

	grub_menu_execute_entry(...)
	{
	  ...
	  /* 执行完 menu entry 中的所有命令，终于可以启动 OS 了。通过执行 "boot" 命令对应的函数
	   * grub_cmd_boot 来实现，代码在 grub-core/commands/boot.c 中 */
	  if (grub_errno == GRUB_ERR_NONE && grub_loader_is_loaded ())
	    /* Implicit execution of boot, only if something is loaded.  */
	    grub_command_execute ("boot", 0, 0);
	}

grub_cmd_boot 函数的内容只有一行：调用 grub_loader_boot 函数，继续看代码：

	grub_err_t grub_loader_boot (void)
	{
	  ...
	  /* 有些模块需注册一些函数在 OS 启动前执行，保存在 preboots_head 中，若有，则依次执行。*/
	  for (cur = preboots_head; cur; cur = cur->next)
	  {
	    ...
	  }
	  /* 执行启动 OS 的 callback 函数。本例中是 grub_linux16_boot */
	  err = (grub_loader_boot_func) ();

	  /* 有些模块需要注册一些函数在 OS 启动后执行，保存在 preboots_tail 中，若有则依次执行 */
	  for (cur = preboots_tail; cur; cur = cur->prev)
	}

在 linux16 命令中, *grub_loader_boot_func* 已被赋值为 *grub_linux16_boot*：

	static grub_err_t grub_linux16_boot (void)
	{
	  grub_uint16_t segment;
	  struct grub_relocator16_state state;

	  /* 以 linux kernel real mode 部分被加载的地址作为段基址，此时就需要类型是整数的地址
	   * 变量 grub_linux_real_target。cs 的值加了 0x20，ip = 0，说明 cpu 将从 real mode
	   * 部分 offset 为 0x200，即 512 bytes 处开始执行代码 */
	  segment = grub_linux_real_target >> 4;
	  state.gs = state.fs = state.es = state.ds = state.ss = segment;
	  state.sp = GRUB_LINUX_SETUP_STACK;
	  state.cs = segment + 0x20;
	  state.ip = 0;
	  state.a20 = 1;

	  grub_video_set_mode ("text", 0, 0);

	  grub_stop_floppy ();

	  /* 重点落到这个函数，下面继续分析*/
	  return grub_relocator16_boot (relocator, state);
	}

	grub_err_t grub_relocator16_boot (..., struct grub_relocator16_state state)
	{
	  /* Put it higher than the byte it checks for A20 check.  */
	  /* 从[0x8010 ~ 0xa0000- RELOCATOR_SIZEOF (16)- GRUB_RELOCATOR16_STACK_SIZE]中
	   * 分配一块 size 为 RELOCATOR_SIZEOF (16) + GRUB_RELOCATOR16_STACK_SIZE 的内存，
	   * 保存在变量 ch 中。在此上下文中，用 A1 表示这块内存的起始地址。RELOCATOR_SIZEOF (16)
	   * 表示 relocator16.S 中 grub_relocator16_end - grub_relocator16_start 的大小。*/
	  err = grub_relocator_alloc_chunk_align (rel, &ch, 0x8010,
					  0xa0000 - RELOCATOR_SIZEOF (16)
					  - GRUB_RELOCATOR16_STACK_SIZE,
					  RELOCATOR_SIZEOF (16)
					  + GRUB_RELOCATOR16_STACK_SIZE, 16,
					  GRUB_RELOCATOR_PREFERENCE_NONE,
					  0);

	  /* 用入参 state 对 relocator16.S 中的各种变量赋值。代码省略 */
	  ...
	  /* 然后将 relocator16.S 中的代码拷贝到刚刚分配的内存 A1 处，等待被跳转执行。
	   * 跳转发生在下面 relst 的那行代码中 */
	  grub_memmove (get_virtual_current_address (ch), &grub_relocator16_start,
					RELOCATOR_SIZEOF (16));

	  /* 又是重点函数，在下面分析 */
	  err = grub_relocator_prepare_relocs (rel, get_physical_target_address (ch),
	                                       &relst, NULL);

	  /* 执行 relocator16.S 的代码拷贝 */
	  ((void (*) (void)) relst) ();
	}

	/* 此函数比较复杂，目前只略看懂了主干 */
	grub_err_t grub_relocator_prepare_relocs (..., grub_addr_t addr, void **relstart,...)
	{
	  /* 通过 malloc_in_range 在[0 - 4G]范围中分配 size 为 7(x86) 或者 12(x86_64) bytes
	   * 的一块内存，然后记录在变量 rels 和 rels0 中。*/
	  ...
	  /* jumper 函数，顾名思义，在刚分配的内存中 hardcode 几条指令进行跳转。由函数里的注释可知，
	   * 写了 2 条指令，对 i386 来说是：
	   *   movl imm32, %eax // 这个立即数是入参 addr，即 A1
	   *   jmp $eax  // 跳到入参 addr 表示的地址处，即 relocator16.S 中 grub_relocator16_start
	   */
	  grub_cpu_relocator_jumper ((void *) rels, (grub_addr_t) addr);
	  /* 将 hardcode 指令的地址传出去，等待执行 */
	  *relstart = rels0;
	}

relocator16.S 的主要工作是从 protect mode 切换回 real mode，步骤可以参考：[Switching from Protected Mode to Real Mode](https://wiki.osdev.org/Real_Mode#Switching_from_Protected_Mode_to_Real_Mode)。代码流程大致如上文所述。它的代码作为 relocator module 的一部分已在内存中，但此时执行的是其在内存中的一份拷贝。下面分析 relocator16.S 的重点代码：

	#include "relocator_common.S"

	VARIABLE(grub_relocator16_start)
		PREAMBLE

PREAMBLE 是定义在 grub-core/lib/i386/relocator_common.S 的宏：

		.macro PREAMBLE
	LOCAL(base):
		/* %rax contains now our new 'base'.  */
		/* local label 'base' 本来是有自己的地址，但是此刻执行的是被 copy 到 A1(eax) 处
		 * 的代码，所以说 eax(x86) 寄存器包含的是新的 'base'。保存 A1 到 esi(x86) 寄存器 */
		mov	RAX, RSI
		...
		/* 加上这个宏定义代码的 size 到 A1 处 */
		add	$(LOCAL(cont0) - LOCAL(base)), RAX
		...
		/* 又一个 absolute jump，跳转到 relocator16.S 中 PREAMBLE 之后的代码处 */
		jmp	*RAX
	LOCAL(cont0):
		.endm

回到 relocator16.S：

		/* 还原 eax 的值，即 A1。此刻 eax, esi 中都是 A1 的值 */
		movl 	%esi, %eax
		/* 用 eax 的值来填充 GDT 中的一项。因为 eax 中的地址 A1 是在 1M 以下，所以 eax 中
		 * 最高的那个 byte 是 0，所以下面的代码只保存低 3 bytes 的内容到 descriptor，所以
		 * 相应的 label 名字叫 cs_base_bytes12 和 cs_base_byte3 */
		movw	%ax, (LOCAL (cs_base_bytes12) - LOCAL (base)) (RSI, 1)
		shrl	$16, %eax
		movb	%al, (LOCAL (cs_base_byte3) - LOCAL (base)) (RSI, 1)

		RELOAD_GDT
		...

RELOAD_GDT 也是定义在 grub-core/lib/i386/relocator_common.S 的宏：

		.macro RELOAD_GDT
		/* 将此宏结束位置(relocator16.S 中)的 effective address(段内offset) 保存到 eax；
		 * 然后继续保存到 local label: jump_vector 处。这里的 (RSI, 1) 是 AT&T 汇编语法，
		 * 参考 "9.15.7 Memory References" of `info as`。有一个 tips：这里的 1 是元素
		 * SCALE，因为手册中有说：BASE 和 INDEX 是寄存器。*/
		lea	(LOCAL(cont1) - LOCAL(base)) (RSI, 1), RAX
		movl	%eax, (LOCAL(jump_vector) - LOCAL(base)) (RSI, 1)

		/* 把 local label: gdt 的 effective address 保存到 eax，再保存到 gdt_addr 处*/
		lea	(LOCAL(gdt) - LOCAL(base)) (RSI, 1), RAX
		mov	RAX, (LOCAL(gdt_addr) - LOCAL(base)) (RSI, 1)

		/* Switch to compatibility mode. GDTR 的内容在下面。 */
		lgdt	(LOCAL(gdtdesc) - LOCAL(base)) (RSI, 1)

		/* Update %cs. 实际只是跳转到本宏结束的位置，这里多此一举的用 long jump 的原因如
		 * 注释所说：更新 cs，因为刚更新完GDT。关于 ljmp 指令的解释，参考 intel 指令手册中
		 * JMP 指令： “Far Jumps in Real-Address or Virtual-8086 Mode.” 和 “Far
		 * Jumps in Protected Mode” 两段。然后此处又是一个 absolute jump，怪不得上面用
		 * lea 指令在 jump_vector 处填入 effective address */
		ljmp	*(LOCAL(jump_vector) - LOCAL(base)) (RSI, 1)

		.p2align	4 /* 以 16 byte 对齐 */
	/* 下面 2 个 label 表示 GDTR 的内容，用于 lgdt 加载时使用。用 2 个 label 表示，
	 * 是因为 gdt_addr 需要被单独赋值 */
	LOCAL(gdtdesc):
		.word	LOCAL(gdt_end) - LOCAL(gdt)
	LOCAL(gdt_addr):
		/* Filled by the code. */
		.long	0

		.p2align	4
	LOCAL(jump_vector):
		/* Jump location. Is filled by the code */
		/* 待加载进 CS 的 segment selector value 定义为 8，表示 GDT 中 index 为 1 的
	     * descripter。参考 intel SDM 3a, Figure 3-6 Segment Selector. 一事不明：
	     * 为什么用 .long 定义 CS selector value？ 咨询社区后得到答复：
	     *     https://www.mail-archive.com/grub-devel@gnu.org/msg27434.html
	     * 看起来应该是个类似手误的问题，多出了 2 个 byte，并不妨事
	     */
		.long	0
		.long	CODE_SEGMENT
	LOCAL(cont1):
		.endm

RELOAD_GDT 后的几行代码给其他 segment register 赋值 DATA_SEGMENT(0x10)，即 GDT 中 index 为 2 的 descriptor。然后：

	DISABLE_PAGING

又是定义在 grub-core/lib/i386/relocator_common.S 的宏：

	.macro DISABLE_PAGING
	movl	%cr0, %eax
	andl	$(~GRUB_MEMORY_CPU_CR0_PAGING_ON), %eax
	movl	%eax, %cr0
	.endm

DISABLE_PAGING 顾名思义，不用过多解释。因为 grub kernel 运行在 protect mode，如果使用 16-bit boot protocol，则需要回到 real mode，才能跳转到 Linux kernel 的 setup 代码。

然后更新所有 segment register，cs 处理特殊一点点：

		/* 更新除 cs 外的其他 segment register。本段代码中，这些寄存器被更新了 2 次，但又
		 * 没有实际内存操作，也得到答复：
		 *     https://www.mail-archive.com/grub-devel@gnu.org/msg27434.html
		 * 老代码就是这样，没人愿意冒风险去动它。
		 */
		...
		/* esi 寄存器一直保存着地址 A1，右移 4 bit，得到 real mode 下的段基址，
		 * 保存到 local label: segment 中。*/
		movl 	%esi, %eax
		shrl	$4, %eax
		movw	%ax, (LOCAL (segment) - LOCAL (base)) (%esi, 1)

		/* 加载 IDTR 的值*/
		lidt (EXT_C(grub_relocator16_idt) - LOCAL (base)) (%esi, 1)

		/* jump to a 16 bit segment. 直接长跳转到 GDT 中定义的 real mode segment。*/
		ljmp	$PSEUDO_REAL_CSEG, $(LOCAL (cont2) - LOCAL(base))

	LOCAL(cont2):
		.code16

		/* clear the PE bit of CR0。这就很明显了，回到 real mode */
		movl	%cr0, %eax
		andl 	$(~GRUB_MEMORY_CPU_CR0_PE_ON), %eax
		movl	%eax, %cr0

		/* flush prefetch queue, reload %cs。因为之前已经填充 label: segment 为段基址，
		 * 而当前 cs 的值是 selector value，所以需要更新为 real mode 下的段基址 */
		/* ljmp。hardcode ljmp 指令 */
		.byte	0xea
		.word 	LOCAL(cont3)-LOCAL(base)
	LOCAL(segment):
		.word	0

	LOCAL(cont3):
		/* 从此开始，工作在真正的 real mode 下*/
		...
		/* 初始化 stack，因为下面 a20 检查的代码中多次出现 call 指令。stack 和代码段在同
		 * 一个 segment 中，因为 label: base 位于这个 segment 的地址 0 处，所以两个
		 * label 之差虽是 offset，但值可以作为地址。然后初始化其 size 为 4k，这也是在
		 * grub_relocator16_boot 函数中被 malloc 过的。 */
		movw    %cs, %ax
		movw    %ax, %ss
		leaw    LOCAL(relocator16_end) - LOCAL(base), %sp
		addw    $GRUB_RELOCATOR16_STACK_SIZE, %sp

		/* 此处略过一堆 a20 检查的代码，又回到重点代码，拿最开始 grub_linux16_boot 函数中
		 * 初始化的各寄存器值，填入到各寄存器中。同样是 hardcode 指令 */
	LOCAL(gate_a20_done):
		/* we are in real mode now
		 * set up the real mode segment registers : DS, SS, ES
		 */
		/* movw imm16, %ax.  */
		.byte	0xb8
	VARIABLE(grub_relocator16_ds)
		.word	0
		movw	%ax, %ds

		/* 下面是其他寄存器的赋值，大部分代码逻辑跟上面一样，故省略。只有 cs 寄存器的更新不同，
		 * 是通过 ljmp 指令，相应的 label 已在 grub_linux16_boot 中初始化 */
		...
		/* ljmp */
		.byte	0xea
	VARIABLE(grub_relocator16_ip)
		.word	0
	VARIABLE(grub_relocator16_cs)
		.word	0

	/* Finally total finished! 终于跳转到内存中已加载的 linux kernel 的 setup 代码。
	 * 回顾上面 CS 寄存器的变化过程：CS 本来是使用的 grub 自己的 GDT 中的 code segment；
	 * 走到 relocator16.S 后，更新为一个临时的由变量 gdt 表示的 GDT 中的 64k 大小的代码段，
	 * 因为它是 protect mode 下的 64k 大小的 segment，所以被称为 PSEUDO_REAL_CSEG，这是
	 * 为跳回 real mode 做准备；以 relocator16.S 所在的起始地址为段基址，又做了一次跳转，
	 * 更新 CS，现在的 CS 和刚刚 protect mode 下 PSEUDO_REAL_CSEG 是同样的段基址；最后
	 * 以加载的 linux kernel 所在地址做段基址，通过 ljmp 更新 cs*/

grub 启动的代码终于结束了，下面进入到 linux kernel，在分析 linux kernel 之前，有必要了解另一个主题： grub 的安装，才能对上面 grub 流程中的部分细节有更确切的理解。

## Install GRUB

安装 grub，需要系统中已安装 grub utility，然后通过 grub-install 将 grub 安装到驱动器中(硬盘或者软盘)。通常只需要指定安装的目标驱动器，比如，通常我们的电脑上只有一块硬盘，叫做 /dev/sda，则只需：

	grub-install /dev/sda [-v]

选项 -v 用来输出安装过程中的详细信息。想要更详细的输出信息，可以使用 `-vv`

官方文档中[科普](https://www.gnu.org/software/grub/manual/grub/html_node/Installation.html#Installation)了一些基础概念：

>GRUB comes with boot images, which are normally put in the directory /usr/lib/grub/`<cpu>`-`<platform>` (for BIOS-based machines /usr/lib/grub/i386-pc). Hereafter, the directory where GRUB images are initially placed will be called the image directory, and the directory where the boot loader needs to find them (usually /boot) will be called the boot directory. 

查看 image directory 会发现安装 grub 所需的所有东西都在这里。

grub-install 的核心内容是：调用 grub-mkimage 生成 core.img，再调用 grub-bios-setup 安装 core.img 和 boot.img，通过 grub-install 的 `-v` 选项可以看出这一点。在我的测试环境(Fedora Workstation 27)中，`-v` 的输出中有如下两条：

>grub-mkimage --directory '/usr/lib/grub/i386-pc' --prefix '(,msdos1)/grub2' --output '/boot/grub2/i386-pc/core.img'  --dtb '' --format 'i386-pc' --compression 'auto'  'ext2' 'part_msdos' 'biosdisk'

>grub-bios-setup  --verbose     --directory='/boot/grub2/i386-pc' --device-map='/boot/grub2/device.map' '/dev/sda'

grub-mkimage 的作用只是生成 core.img，虽然它的 man page 没有明确说明。由上面的图示可知 core.img = diskboot.img[1] + lzma_decompress.img[2] + kernel.img + `<mods>`

1. diskboot.img 用于硬盘启动的环境下，对于其他的情况有不同的 image，比如 CDROM 有 cdboot.img，网络启动，有 pxeboot.img。
2. 由于 kernel.img 和 mods 一起被压缩，所以必须有相应的解压缩代码，不同的压缩算法对应了不同的 decompress image，对于 x86，默认是 lzma。

下面我们分别分析 grub-mkimage 和 grub-bios-setup 的细节，再回归到 grub-install。

#### grub-mkimage

grub-mkimage 的源代码在 util/grub-mkimage.c 中，代码结构比较清晰，解析检查命令行入参后，调用函数 *grub_install_generate_image* 生成 image，下面分析此函数的大致过程。

先是读取 kernel.img 和相关模块，打包在一起并压缩，如果不想看下面的代码，可以直接跳到下面的图示感受下格式即可。

	struct grub_mkimage_layout layout;
	...

	/* grub-mkimage 命令行必须传入需要添加的 modules，所以首先读取 moddep.lst 获得依赖关系，
	 * 将传入的 module 及其依赖的 module 列出到 *path_list*，并读取获得总 size */
	path_list = grub_util_resolve_dependencies (dir, "moddep.lst", mods);
	...
	for (p = path_list; p; p = p->next)
	total_module_size += (ALIGN_ADDR (grub_util_get_image_size (p->name))
			  + sizeof (struct grub_module_header));
	...

	/* 读取解析 kernel.img。解析后的相关信息放在变量 *layout* 中。
	 * kernel_img 指向包含 kernel.img 和 modules 的 buffer */
	kernel_path = grub_util_get_path (dir, "kernel.img");
	...
	if (image_target->voidp_sizeof == 4)
	  kernel_img = grub_mkimage_load_image32 (kernel_path, total_module_size,
					    &layout, image_target);
	else
	  kernel_img = grub_mkimage_load_image64 (kernel_path, total_module_size,
					    &layout, image_target);

函数 grub_mkimage_load_image32/grub_mkimage_load_image64 的代码在 util/grub-mkimagexx.c 中：

	char *
	SUFFIX (grub_mkimage_load_image) (const char *kernel_path,
				  size_t total_module_size,
				  struct grub_mkimage_layout *layout,
				  const struct grub_install_image_target_desc *image_target)

SUFFIX 宏分别定义在 util/grub-mkimage32.c：

	# define SUFFIX(x)	x ## 32

和 util/grub-mkimage64.c：

	# define SUFFIX(x)	x ## 64

下面以 IA32 代码为例介绍

	/* 获取模块空间的起始地址，放在 modinfo 中*/
	modinfo = (struct grub_module_info32 *) (kernel_img + layout.kernel_size);

	/* modinfo 是个简单的 structure，初始化 */
	modinfo->magic = grub_host_to_target32 (GRUB_MODULE_MAGIC);
	modinfo->offset = grub_host_to_target_addr (sizeof (struct grub_module_info32));
	modinfo->size = grub_host_to_target_addr (total_module_size);

	offset = layout.kernel_size + sizeof (struct grub_module_info32);

	/* 依次读取模块，并给每一个添加 header，放入模块空间中 */
	for (p = path_list; p; p = p->next)
	{
	  struct grub_module_header *header;
	  size_t mod_size;

	  mod_size = ALIGN_ADDR (grub_util_get_image_size (p->name));

	  header = (struct grub_module_header *) (kernel_img + offset);
	  header->type = grub_host_to_target32 (OBJ_TYPE_ELF);
	  header->size = grub_host_to_target32 (mod_size + sizeof (*header));
	  offset += sizeof (*header);

	  grub_util_load_image (p->name, kernel_img + offset);
	  offset += mod_size;
	}
	...
	/* 将 kernel.img 和 modules 合在一起并压缩，压缩后由 core_img 表示，大小是 core_size */
	compress_kernel (image_target, kernel_img, layout.kernel_size + total_module_size,
		   &core_img, &core_size, comp);

压缩前的数据长这样：

![kernel & mod](grub_kern_mod.png)

	/* 对于 i386-pc 来说，默认压缩方式是 GRUB_COMPRESSION_LZMA(参考全局变量 image_targets)，
	 * 选择并读取相应的解压缩 image */
	decompress_path = grub_util_get_path (dir, name);
	decompress_size = grub_util_get_image_size (decompress_path);
	decompress_img = grub_util_read_image (decompress_path);

在前面小节中已提到，**lzma_decompress.img** 中 special data 中的一些数据要由 grub-mkimage 写入，这是其中两个:

	/* 将 compressed data 压缩前后的 size 分别写入相应的位置 */
	if (image_target->decompressor_compressed_size != TARGET_NO_FIELD)
	  *((grub_uint32_t *) (decompress_img + image_target->decompressor_compressed_size))
					= grub_host_to_target32 (core_size);

	if (image_target->decompressor_uncompressed_size != TARGET_NO_FIELD)
	  *((grub_uint32_t *) (decompress_img + image_target->decompressor_uncompressed_size))
					= grub_host_to_target32 (layout.kernel_size + total_module_size);

将压缩后的 kernel.img + mods 和 lzma_decompress.img 先 copy 到一个 buffer:

	full_size = core_size + decompress_size;
	full_img = xmalloc (full_size);
	memcpy (full_img, decompress_img, decompress_size);
	memcpy (full_img + decompress_size, core_img, core_size);

	free (core_img);
	/* 继续用变量 core_img 和 core_size 表示 */
	core_img = full_img;
	core_size = full_size;
	free (decompress_img);
	free (decompress_path);

最后，对于硬盘启动的情况来说，需要将 diskboot.img 添加到 lzma_decompress.img 之前，并修改 diskboot.img 结尾的blocklist 数据结构，它才能知道去哪儿寻找加载 core.img。

	/* 将 lzma_decomress.img + kernel.img + <mods> 的 size 转换为 sector number */
	num = ((core_size + GRUB_DISK_SECTOR_SIZE - 1) >> GRUB_DISK_SECTOR_BITS);

	/* 找到 diskboot.img */
	boot_path = grub_util_get_path (dir, "diskboot.img");
	boot_size = grub_util_get_image_size (boot_path);
	...
	boot_img = grub_util_read_image (boot_path);
	{
	  /* 在 diskboot.img 一节说过，image 的最后面是 blocklist 数据结构。blocklist 的
	   * len 字段表示 core.img 中除 diskboot.img 以外的 size(sector 为单位)。只写入
	   * size，起始地址在安装的时候才知道 */
	  struct grub_pc_bios_boot_blocklist *block;
	  block = (struct grub_pc_bios_boot_blocklist *) (boot_img
							  + GRUB_DISK_SECTOR_SIZE
							  - sizeof (*block));
	  block->len = grub_host_to_target16 (num);
	  ...
	}
	/* 先将 diskboot.img 写入最终的 core.img 的 buffer 中 */
	grub_util_write_image (boot_img, boot_size, out, outname);
	...
	/* 再将剩余的内容写入 */
	grub_util_write_image (core_img, core_size, out, outname);

core.img 的生成过程就是这样简单。

#### grub-bios-setup

man 手册中说：

>Set up images to boot from a device. You should not normally run this program directly.  Use grub-install instead.

正如上文中提到，在 `grub-install -v` 的输出中有：
>grub-bios-setup  --verbose     --directory='/boot/grub2/i386-pc' --device-map='/boot/grub2/device.map' '/dev/sda'

没有指定 boot image 和 core image 的时候，默认是 `--directory` 下的 boot.img 和 core.img。

它的主要作用是将 core.img 和 boot.img 写入磁盘，源代码在 util/grub-setup.c，我们拣关键代码分析。和其他 grub utility 一样，开始是入参解析，然后作一堆初始化动作，看起来是为了访问 /boot 所在文件系统所需，最重要的代码就只有这个函数：

	/* Do the real work.  */
	GRUB_SETUP_FUNC (arguments.dir ? : DEFAULT_DIRECTORY,
		   arguments.boot_file ? : DEFAULT_BOOT_FILE,
		   arguments.core_file ? : DEFAULT_CORE_FILE,
		   dest_dev, arguments.force,
		   arguments.fs_probe, arguments.allow_floppy,
		   arguments.add_rs_codes);

奇妙的是，这个 GRUB_SETUP_FUNC 定义在根目录的 Makefile 中：

	grub_bios_setup_CPPFLAGS = $(AM_CPPFLAGS) $(CPPFLAGS_PROGRAM) -DGRUB_SETUP_FUNC=grub_util_bios_setup

但直接搜 grub_util_bios_setup 也找不到，原来定义在 util/setup.c 中：

	#ifdef GRUB_SETUP_BIOS
	#define SETUP grub_util_bios_setup
	#elif GRUB_SETUP_SPARC64
	#define SETUP grub_util_sparc_setup
	#else
	#error "Shouldn't happen"
	#endif

	void
	SETUP (const char *dir,
	       const char *boot_file, const char *core_file,
	       const char *dest, int force,
	       int fs_probe, int allow_floppy,
	       int add_rs_codes __attribute__ ((unused))) /* unused on sparc64 */
	{
	  ...
	}

下面来看函数 grub_util_bios_setup 的内容：

	struct blocklists bl;

	bl.first_sector = (grub_disk_addr_t) -1;

	/* bl.current_segment = 0x8200，是 lzma_decompress.img 被编译 & 加载的地址 */
	#ifdef GRUB_SETUP_BIOS
		bl.current_segment =
		    GRUB_BOOT_I386_PC_KERNEL_SEG + (GRUB_DISK_SECTOR_SIZE >> 4);
	#endif
	bl.last_length = 0;

	/* 读取 boot.img 和 core.img */
	boot_path = grub_util_get_path (dir, boot_file);
	boot_size = grub_util_get_image_size (boot_path);
	boot_img = grub_util_read_image (boot_path);

	core_path = grub_util_get_path (dir, core_file);
	core_size = grub_util_get_image_size (core_path);
	core_img = grub_util_read_image (core_path);

	/** 将 core.img 的 size 转换成 sector，即除 512 */
	#ifdef GRUB_SETUP_BIOS
		core_sectors = ((core_size + GRUB_DISK_SECTOR_SIZE - 1)
			  >> GRUB_DISK_SECTOR_BITS);
	#endif

	/* 获取 diskboot.img 结尾的 blocklist 数据结构的地址 */
	bl.first_block = (struct grub_boot_blocklist *) (core_img
				   + GRUB_DISK_SECTOR_SIZE
				   - sizeof (*bl.block));
	...
	/* 打开目的设备，即 /dev/sda */
	dest_dev = grub_device_open (dest);
	...
	/* 省略一段 root device 分析，因为目前不确定它的含义。目前的理解是：一台 PC 上可能有很多
	 * 块磁盘，安装了 grub 或操作系统那一块才叫 root device? 那么一般 PC 只有一块硬盘，它
	 * 就是 root device。艰难的看了这段代码，理解如下：从 /proc/self/mountinfo 中读取信息，
	 * 跟入参 dir: /boot/grub/i386-pc 比对，找到 mountinfo 中 mount point 是 /boot 的
	 * mount source(参考 man proce)，最后确定 root device, 并设置到环境变量。我的测试中
	 * 显示 root = hostdisk//dev/sda,msdos1 */
	grub_util_info ("setting the root device to `%s'", root);
	if (grub_env_set ("root", root) != GRUB_ERR_NONE)
		grub_util_error ("%s", grub_errmsg);

	#ifdef GRUB_SETUP_BIOS
	  {
		/* Read the original sector from the disk.  */
		/* 读取当前 boot sector 中的内容，作用是：
		 *   1. copy 当前 boot sector 可能存在的 BPB 数据;
		 *   2. 修改指令适配有问题的 BIOS; 3. copy 分区表
		 */
		tmp_img = xmalloc (GRUB_DISK_SECTOR_SIZE);
		if (grub_disk_read (dest_dev->disk, 0, 0, GRUB_DISK_SECTOR_SIZE, tmp_img))
		  grub_util_error ("%s", grub_errmsg);

		boot_drive_check = (grub_uint8_t *) (boot_img
						  + GRUB_BOOT_MACHINE_DRIVE_CHECK);
		/* Copy the possible DOS BPB.  */
		memcpy (boot_img + GRUB_BOOT_MACHINE_BPB_START,
	    tmp_img + GRUB_BOOT_MACHINE_BPB_START,
	    GRUB_BOOT_MACHINE_BPB_END - GRUB_BOOT_MACHINE_BPB_START);
		/* 上述源码注释已写的很清楚，无需赘述 */

		/* If DEST_DRIVE is a hard disk, enable the workaround, which is
		   for buggy BIOSes which don't pass boot drive correctly. Instead,
		   they pass 0x00 or 0x01 even when booted from 0x80.  */
		/* 上面的注释把 bug 解释的很清楚，关于 boot drive number 的问题在 boot.img 一节
		 * 已有解释。看起来一般情况下都会改写 boot.img 中 boot_drive_check 处的 jmp 指令
		 */
		if (!allow_floppy && !grub_util_biosdisk_is_floppy (dest_dev->disk))
		{
			/* Replace the jmp (2 bytes) with double nop's.  */
			boot_drive_check[0] = 0x90;
			boot_drive_check[1] = 0x90;
		}
		...
		/* 获得 core image 将被安装到的磁盘位置(sector 号)，存放在数组 sectors */
		if (is_ldm)
		  err = grub_util_ldm_embed (dest_dev->disk, &nsec, maxsec,
				 GRUB_EMBED_PCBIOS, &sectors);
		else if (ctx.dest_partmap)
		  err = ctx.dest_partmap->embed (dest_dev->disk, &nsec, maxsec,
				     GRUB_EMBED_PCBIOS, &sectors);
		else
		  err = fs->embed (dest_dev, &nsec, maxsec,
		         GRUB_EMBED_PCBIOS, &sectors);

		/* 清零原 diskboot.img 中 blocklist 结构体 */
	    /* Clean out the blocklists.  */
	    bl.block = bl.first_block;
	    while (bl.block->len)
		{
			grub_memset (bl.block, 0, sizeof (*bl.block));

			bl.block--;

			if ((char *) bl.block <= core_img)
			grub_util_error ("%s", _("no terminator in the core image"));
		}
		/* 在此函数中更新 diskboot.img 中的 blocklist 数据，细节略过 */
		save_blocklists (sectors[i] + grub_partition_get_start (ctx.container),
		       0, GRUB_DISK_SECTOR_SIZE, &bl);

		/* 已经确定 core.img 将要安装的 sector 地址，也要更新 boot.img 中的相关字段，使
		 * boot.img 可以找到 core.img 第一个 sector 的内容，也即 diskboot.img */
		write_rootdev (root_dev, boot_img, bl.first_sector);
		...

		/* 将 core.img 写入 */
		/* Write the core image onto the disk.  */
	    for (i = 0; i < nsec; i++)
		  grub_disk_write (dest_dev->disk, sectors[i], 0,
			       GRUB_DISK_SECTOR_SIZE,
			       core_img + i * GRUB_DISK_SECTOR_SIZE);

		/* 最后写入 boot.img */
		/* Write the boot image onto the disk.  */
	    if (grub_disk_write (dest_dev->disk, BOOT_SECTOR,
			       0, GRUB_DISK_SECTOR_SIZE, boot_img))
		    grub_util_error ("%s", grub_errmsg);


#### grub-install
介绍完了 grub-mkimage, grub-bios-setup，回头看 grub-install 的工作流程。上面已经说过，通常我们安装 grub 时候，只需要

	grub-install /dev/sda

而无需指定额外参数，grub 会自动配置好其他参数。grub-install 的工作主要是将 grub 的文件从 image directory 拷贝到 boot directory，然后调用 grub-mkimage 生成 core.img，调用 grub-bios-setup 来安装 boot.img 和 core.img。我们仍选择关键代码分析，以理解其过程。

	/* 首先是一堆准备工作，各种读取文件*/
	/* grub_install_source_directory 便是上文说的 image directory，target 在我们的
	 * 例子中是 "i386-pc" */
	if (!grub_install_source_directory)
	  {
	    if (!target)
		  {
		    const char * t;
		    t = get_default_platform ();
			if (!t)
			  grub_util_error ("%s",
			     _("Unable to determine your platform."
			       " Use --target."));
			  target = xstrdup (t);
		  }
		grub_install_source_directory
			= grub_util_path_concat (2, grub_util_get_pkglibdir (), target);
	  }

	/* 通过 string 来获得枚举变量 platform。括号里面的代码看起来仅仅是 debug 用 */
	platform = grub_install_get_target (grub_install_source_directory);

	{
	  char *platname = grub_install_get_platform_name (platform);
	  fprintf (stderr, _("Installing for %s platform.\n"), platname);
	  free (platname);
	}

	switch (platform)
	{
	  /* 对于 i386-pc 平台来说，默认访问磁盘的方式是通过 BIOS INT13，所以 disk_module
	   * 是 biosdisk module */
  	  case GRUB_INSTALL_PLATFORM_I386_PC:
  	    if (!disk_module)
		  disk_module = xstrdup ("biosdisk");
	    break;
	  ...
	}

	/* 创建 grub directory，一般是 /boot/grub/，可以在 configure 阶段进行配置 */
	if (!bootdir)
	  bootdir = grub_util_path_concat (3, "/", rootdir, GRUB_BOOT_DIR_NAME);

	{
	  char * t = grub_util_path_concat (2, bootdir, GRUB_DIR_NAME);
	  grub_install_mkdir_p (t);
	  grubdir = grub_canonicalize_file_name (t);
	  if (!grubdir)
	    grub_util_error (_("failed to get canonical path of `%s'"), t);
	  free (t);
	}

	/* 对于非 efi 来说，直接来到下面的函数，将 image directory 下的内容 copy 到 boot
	 * directory。copy 的内容包括：
	 *   1. image directory 下所有的 .mod 文件；
	 *   2. {"efiemu32.o", "efiemu64.o", "moddep.lst", "command.lst", "fs.lst",
	 *       "partmap.lst", "parttool.lst", "video.lst", "crypto.lst", "terminal.lst",
	 *       "modinfo.sh"} 等 12 个文件；
	 *   3. image directory 下的 po 目录，即 locale 文件(在我们的测试环境中没有 po 目录)；
	 *   4. /usr/lib/grub/theme(我们的测试下环境) 下的 theme 文件; /usr/lib/grub/
	 *      下的 fonts 文件(.pf2)
	 */
	grub_install_copy_files (grub_install_source_directory,
							 grubdir, platform);

	/* 跨过一堆磁盘/文件系统检测相关的代码，终于来待生成 core.img 的代码 */
	/* 不得不说，grub 的代码写的有点乱。这里的 platdir 指 boot directory，即
	 * /boot/grub/i386-pc，所以 imgfile 指 /boot/grub/i386-pc/core.img
	 */
	char *imgfile = grub_util_path_concat (2, platdir, core_name);
	char *prefix = xasprintf ("%s%s", prefix_drive ? : "", relative_grubdir);

	/* 调用 grub-mkimage 命令生成 core.img，其实调用的是该命令的主要函数：
	 * grub_install_generate_image */
	grub_install_make_image_wrap (/* source dir  */ grub_install_source_directory,
				/*prefix */ prefix,
				/* output */ imgfile,
				/* memdisk */ NULL,
				have_load_cfg ? load_cfg : NULL,
				/* image target */ mkimage_target, 0);

	/* core.img 已经生成到 boot directory 中，完成了大约一半的工作。下面该把 boot.img
	 * 也放到里面，过程比较简单，仅是从 image directory 中 copy 到 boot directory，即:
	 * /usr/lib/grub/i386-pc/boot.img --> /boot/grub/i386-pc/boot.img
	 */
	{
	  char *boot_img_src = grub_util_path_concat (2,
		  				  grub_install_source_directory,
						  "boot.img");
	  char *boot_img = grub_util_path_concat (2, platdir,
					      "boot.img");
	  grub_install_copy_file (boot_img_src, boot_img, 1);
	  /*  Now perform the installation. */
	  /* install_bootsector 默认是 1，所以默认将调用 grub-bios-setup 的主函数来安装
	   * boot.img 和 core.img */
	  if (install_bootsector)
	    grub_util_bios_setup (platdir, "boot.img", "core.img",
				  install_drive, force,
				  fs_probe, allow_floppy, add_rs_codes);
	}

这就是 grub-install 的过程，大部分的代码是为了最后调用 grub-mkimage 和 grub-bios-setup 做准备。

## How linux kernel is booted

linux kernel 编译出来的 bzImage 中包括如下两部分：

 1. 运行在 real mode 下的 arch/x86/boot/setup.bin，在 boot loader 使用 16-bit boot protocol 时才会执行，下文统称之为 **setup**
 2. 运行在 protect mode 或 long mode 下的 arch/x86/boot/vmlinux.bin, 它包含了压缩后的 kernel(vmlinux) 和 relocs（定位信息的), 它的最主要作用是解压缩，所以下文统称之为 **decompressor**

上文中所说 linux kernel 的 real mode 部分即是 setup.bin，位于 linux kernel 的 arch/x86/boot/ 目录。由 grub 的 linux16 命令加载启动的内核将首先执行 setup.bin 的代码 所以，首先来看 setup.bin 的流程.

因为本文分析的是 x86 代码，势必会很多地方引用官方文档: [Intel Software Developer Manual](https://software.intel.com/en-us/articles/intel-sdm), 下文统一用 Intel SDM 代指。Intel SDM 分为若干 volume, 下文将使用 [10 volume](https://software.intel.com/en-us/articles/intel-sdm#nine-volume) 的方式引用: 1,  2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, 4.

### arch/x86/boot/setup.bin

setup.bin 的二进制文件布局由其 linker script arch/x86/boot/setup.ld 定义:

	...
	SECTIONS
	{
	    . = 0;
	    .bstext         : { *(.bstext) }
	    .bsdata         : { *(.bsdata) }

	    . = 495;
	    .header         : { *(.header) }
	    .entrytext      : { *(.entrytext) }
	    .inittext       : { *(.inittext) }
	    .initdata       : { *(.initdata) }
	    __end_init = .;

		...
	}

可知，.bstext 和 .bsdata section 都定义在第一个 sector 即 boot sector 中，bs 即缩写。再由代码 arch/x86/boot/header.S 可知，setup.bin 最开始执行的指令在 header.S 中。linux 诞生之初好像还没有 boot loader，且被安装在软盘中使用，所以自带 boot sector 代码，现在因为向后兼容，依然保留了 boot sector 的内容。

因为 linux 支持 UEFI 启动，UEFI 可以不依赖 boot sector 而直接加载相应的 OS loader 或者 OS kernel，所以 setup.bin 需要伪装成 UEFI 支持的 image format。UEFI spec 中对 UEFI image 的描述：

>2.1.1 UEFI Images

>UEFI Images are a class of files defined by UEFI that contain executable code. The most distinguishing feature of UEFI Images is that the first set of bytes in the UEFI Image file contains an image header that defines the encoding of the executable image.

>UEFI uses a subset of the PE32+ image format with a modified header signature. The modification to the signature value in the PE32+ image is done to distinguish UEFI images from normal PE32 executables. The “+” addition to PE32 provides the 64-bit relocation fix-up extensions to standard PE32 format.

所以，UEFI image 的格式是 PE format，这是 Windows 操作系统下的可执行文件和目标文件等的格式，可以参考：

1. [PE 官方文档](https://docs.microsoft.com/en-us/windows/desktop/debug/pe-format)
2. [PE 32bit illustration](https://commons.wikimedia.org/wiki/File:Portable_Executable_32_bit_Structure.png)
3. [The Linux EFI Boot Stub](https://firmware.intel.com/blog/linux-efi-boot-stub)

用文字简单概括 PE 格式： MS-DOS stub(Image Only) + signature("PE\0\0", Image Only) + COFF file header(Object and Image) + Optional header(Image Only，包括3个主要部分：Standard fields, Windows-specific fields, Data directories) + section table。图示如下：

![PE32](Portable_Executable_32_bit_Structure.png)

下面进行代码详细分析。

#### arch/x86/boot/header.S

		.code16
		.section ".bstext", "ax"

		.global bootsect_start
	bootsect_start:
	#ifdef CONFIG_EFI_STUB
		# "MZ", MS-DOS header
		# 一个 PE 格式的可执行文件开头是 Microsoft MS-DOS stub，这个 stub 前两个 byte
		# 是魔术字 “MZ”。后面是 stub 中的内容，不用关心。
		.byte 0x4d
		.byte 0x5a
	#endif

	...

	#ifdef CONFIG_EFI_STUB
		# 在 dos stub 的偏移 0x3c 处，如 PE spec 中所说：At location 0x3c,
		# the stub has the file offset to the PE signature.
		.org	0x3c
		# Offset to the PE header.
		.long	pe_header
	#endif /* CONFIG_EFI_STUB */

		.section ".bsdata", "a"
		...
	#ifdef CONFIG_EFI_STUB
		# 上文所说的 signature
		pe_header:
		.ascii	"PE"
		.word 	0

	# coff file header 开始，参考 "PE 官方文档"，找代码中每一个字段的出处
	coff_header:
	#ifdef CONFIG_X86_32
		.word	0x14c				# i386
	#else
		.word	0x8664				# x86-64
	#endif
		...

	# optional header 开始，同样对照 "PE 官方文档"
	# 开头是 optional header magic number，然后是 optional header 的 Standard Fields
	optional_header:
	#ifdef CONFIG_X86_32
		.word	0x10b				# PE32 format
	#else
		.word	0x20b 				# PE32+ format
	#endif
		...

	# optional header 的 Windows-Specific Fields。因为是 Windows-Specific，
	# 所以下面的大多 fields 是空的？
	extra_header_fields:
	#ifdef CONFIG_X86_32
		.long	0				# ImageBase
	#else
		.quad	0				# ImageBase
	#endif

	# Optional Header 的 Data Directories
		.quad	0				# ExportTable
		.quad	0				# ImportTable
		...

	# Section table，包含多个 section header entry，其数目由 coff header 中的
	# NumberOfSections 指定。Hardcode 4 个 section 是为啥？
	section_table:
		# The offset & size fields are filled in by build.c.
		# 第一个 field 是 name，8 bytes 长，又因为 .ascii 表示的字符串在汇编时不包含结尾
		# 的空字符，所以，用 2 个 .byte 0 补齐
		.ascii	".setup"
		.byte	0
		.byte	0
		...

		.ascii	".reloc"
		...
		.ascii	".text"
		...
		.ascii	".bss"
		...
	# PE file header 的内容到此结束，但 size 还没有 1 个 sector 大(512 byte)
	#endif //CONFIG_EFI_STUB

	# 由 arch/x86/boot/setup.ld 可知，.header section 从地址 495 开始。
	.section ".header", "a"
		.globl	sentinel
	sentinel:	.byte 0xff, 0xff        /* Used to detect broken loaders */

	# hdr 开始的内容属于 linux/x86 boot protocol，定义在 Documentation/x86/boot.txt
	.globl	hdr
	hdr:
	setup_sects:	.byte 0			/* Filled in by build.c */
	root_flags:	.word ROOT_RDONLY
	syssize:	.long 0			/* Filled in by build.c */
	ram_size:	.word 0			/* Obsolete */
	vid_mode:	.word SVGA_MODE
	root_dev:	.word 0			/* Filled in by build.c */
	boot_flag:	.word 0xAA55
	# 至此，第一个 sector 的内容才完整了，从最后的 0xAA55 也可以发现

		# offset 512, entry point。正如上文对 grub 函数 grub_linux16_boot 的分析一样。
		.globl	_start
	_start:
		# 下面注释说的清楚。但为什么 assembler 会生成 3 bytes 的 jmp 指令？难道因为当前是
		# 16-bit 的 real mode，所以 GCC 会默认生成 rel16 的 displacement？
		# 一般 jmp 指令是相对跳转，即跳转相对当前位置的偏移，所以指令的第二个 byte 表示偏移。
		# start_of_setup 是 .entrytext section 的第一个地址，也即跳转目的地址。

		# Explicitly enter this as bytes, or the assembler
		# tries to generate a 3-byte jump here, which causes
		# everything else to push off to the wrong offset.
		.byte	0xeb		# short (2-byte) jump
		.byte	start_of_setup-1f
	1:

	# 下面的内容依然是 linux boot protocol，所以说是 part 2
	# Part 2 of the header, from the old setup.S。
		.ascii	"HdrS"		# header signature
		.word	0x020d		# header version number (>= 0x0105)
		...
	# 省略 boot protocol 的内容，待有需要时候再进行分析。紧挨着 boot protocol 的就是上面
	# 跳转指令的目的地，在 section .entrytext 中。

	# End of setup header #####################################################
	.section ".entrytext", "ax"
	start_of_setup:
		# Force %es = %ds
		movw	%ds, %ax
		movw	%ax, %es
		cld

	# Apparently some ancient versions of LILO invoked the kernel with %ss != %ds,
	# which happened to work by accident for the old code.  Recalculate the stack
	# pointer if %ss is invalid.  Otherwise leave it alone, LOADLIN sets up the
	# stack behind its own code, so we cant blindly put it directly past the heap.
		movw	%ss, %dx
		cmpw	%ax, %dx	# %ds == %ss?
		movw	%sp, %dx # sp 在 grub 函数 grub_linux16_boot 中已被赋值为 0x9000
		je	2f		# -> assume %sp is reasonably set
		...

	# ds = ss 的正常情况下，跳转至此
	2:	# Now %dx should point to the end of our stack space
		# dx 此时的值是 sp = 0x9000。 and 操作强制抹 0 尾部 2 个 bit，为了 dword 对齐
		andw	$~3, %dx	# dword align (might as well...)
		jnz	3f
		# 万一 dx 中是个无效值，我们也要给它赋个有效值，segment 最高的地址，64k 的最大处
		movw	$0xfffc, %dx	# Make sure we're not zero
	3:	movw	%ax, %ss  # ax 中是 ds 的值
		movzwl	%dx, %esp	# Clear upper half of %esp。将校正过的 sp 放回去
		# 恢复响应中断，上次 cli 发生在 grub 的函数 grub_relocator16_boot 中
		sti			# Now we should have a working stack

	# We will have entered with %cs = %ds+0x20, normalize %cs so
	# it is on par with the other segments. 用这个技巧重新 load cs 和 ip 寄存器
		pushw	%ds
		pushw	$6f
		lretw
	6:

	# Check signature at end of setup.
	# setup_sig 是定义在 linker script 中的变量。理论上他们不会不相等，参考 Maintainer
	# 的解释：https://lkml.org/lkml/2018/3/21/226。下面的 bss 清零代码也有个小问题，也在
	# 上面的邮件里得到确认和澄清。
		cmpl	$0x5a5aaa55, setup_sig
		jne	setup_bad

	# Zero the bss
		movw	$__bss_start, %di
		movw	$_end+3, %cx
		xorl	%eax, %eax
		subw	%di, %cx
		shrw	$2, %cx
		rep; stosl

	# Jump to C code (should not return)。跳入 setup 的 C 函数，在 arch/x86/boot/main.c
	# 中。call 指令奇怪的加了 l 后缀，用于指定内存操作数的 size，但在 real mode 中，并不需要
	# l(32 bit)。怀疑是作者手误，改成 "call" 经测试没有问题。看看 setup 的反汇编：
	#	2bb:   66 e8 29 2b             callw  2de8 <bios_probe+0x108>
	#	...
	#	00002dea <main>:
	#	...
	# 这里 call 指令使用的是 relative offset，即相对当前 PC(IP 寄存器)的 offset， 使用
	# l 后缀导致 call 指令机器码前面有前缀 66: Operand-size override prefix。即操作数是
	# 32bit，又因为 x86 是小端，所以其实 objdump 反汇编的机器码中少显示了 2 个 0 byte，也
	# 就是说整个 call 指令是 6 bytes 大小，所以跳转地址是： 2bb + 6 + 2b29 = 2dea，其中
	# (2bb+6) 是当前 IP 的值，2b29是嵌在指令中的相对 offset。

		calll	main

#### arch/x86/boot/main.c

header.S 中最后一条指令跳入了 setup.bin 的 main 函数，终于进入了 C 语言部分。main 函数内容如下，依然分析重点代码

	void main(void)
	{
		/* First, copy the boot header into the "zeropage" */
		copy_boot_params();

		/* Initialize the early-boot console */
		/* 解析 command line 是否有 “console=xxx” 的选项，初始化串口，在 I/O port 地址空间 */
		console_init();
		if (cmdline_find_option_bool("debug"))
			puts("early console in setup code\n");

		/* End of heap check */
		init_heap();

		/* Make sure we have all the proper CPU support */
		/* 看起来并不重要，略过 */
		if (validate_cpu()) {
			...
		}

		/* Tell the BIOS what CPU mode we intend to run in. */
		/* 这是 x86-64 专用代码，其中断: "int 0x15 eax=0xec00, ebx=0x2"，在网上的
		 * bios interrupt list 找不到任何描述。这是目前唯一的发现：
		 *     https://forum.osdev.org/viewtopic.php?f=1&t=20445&start=0
		 */
		set_bios_mode();

		/* Detect memory layout */
		/* 依次通过 int 0x15 ax=E820/E801/88 中断获取 memory map 信息，填充到全局变量
		 * boot_params 的不同的字段中。参考阅读：
		 *     https://wiki.osdev.org/Detecting_Memory_(x86)。
		 * 另外，为使用 real mode 下的 bios 中断服务，专门写了叫 intcall 的汇编函数，
		 * 在 setup 的代码中广泛使用，值的分析，TBD
		 */
		detect_memory();

		/* Set keyboard repeat rate (why?) and query the lock flags */
		/* 也是通过 bios 中断(intcall 函数)来查询键盘信息存在 boot_params 变量中，
		 * 并设置键盘的配置参数 */
		keyboard_init();

		/* Query Intel SpeedStep (IST) information */
		/* 这个更不是重点了，仍然是通过 bios 中断获取信息存在 boot_params 中 */
		query_ist();

		/* 略过两个非重点函数：query_apm_bios，query_edd，set_video，
		 * 过程依然主要是 bios 中断 -> boot_params */

		/* Do the last things and invoke protected mode。这才是重点 */
		go_to_protected_mode();
	}

对重点函数单独分析：

	/* 把 header.S 中 hdr 起始的 boot protocol 的内容 copy 到另一处的全局变量
	 * boot_params 中 */
	static void copy_boot_params(void)
	{
		...
		/* 这个宏很有意思，详细解释在:include/linux/build_bug.h 中。简言之：括号中的条件
		 * 为真时，编译将报错退出。同时引出一个新东西：__OPTIMIZE__，GCC 的 Common
		 * Predefined Macros，参考：
		 *     https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html
		 */
		BUILD_BUG_ON(sizeof boot_params != 4096);
		memcpy(&boot_params.hdr, &hdr, sizeof hdr);

		/* 下面一段代码针对老版本的 boot protocol 做相关处理，无需关注。需要配合 grub 的
		 * grub_cmd_linux 一起看，只需要看字段 cl_offset，setup_move_size 的赋值即可 */
	}

	static void init_heap(void)
	{
		char *stack_end;

		/* gcc inline assembly 的介绍可以参考：
		 *     https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html
		 * 这句内嵌汇编的意思: stack_end = %esp - STACK_SIZE = %esp - 0x400。进入
		 * main 函数前 esp 的值是 0x9000，因为本函数没有入参，所以理论上此时 esp 的值不变，
		 * 待验证。回忆一下：在 grub 为 linux kernel 分配内存时，为了 stack 和 heap 共
		 * 分配了 4k 内存
		 */
		if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
			asm("leal %P1(%%esp),%0"
			    : "=r" (stack_end) : "i" (-STACK_SIZE));

			/* Documentation/x86/boot.txt 对 heap_end_ptr 的定义有减去 0x200，grub
			 * 代码中也是这样做的：给 heap_end_ptr 赋值为 GRUB_LINUX_HEAP_END_OFFSET
			 * = 0x9000 - 0x200，所以这里使用前需要复原一下。
			 */
			heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
			if (heap_end > stack_end)
				heap_end = stack_end; /* stack 优先，4k 中，1k 作 stack, 3k 作 heap */
		} else {
			/* Boot protocol 2.00 only, no heap available */
			puts("WARNING: Ancient bootloader, some functionality "
			     "may be limited!\n");
		}
	}

函数 detect_memory 依次使用 BIOS Function: INT 0x15, AX = 0xE820/0xE801/0x88 来获取内存信息，后两者比较少见，着重分析。

E820 很常见，就不介绍它的用法了。在本代码中使用 E820 中断服务，将获取的内存信息保存到 boot_params.e820_table 中。

对 E801 比较标准的定义是：
>It is built to handle the 15M memory hole, but stops at the next hole / memory mapped device / reserved area above that. That is, it is only designed to handle contiguous memory above 16M. 

它的返回值是：
>AX = CX = extended memory between 1M and 16M, in K (max 3C00h = 15MB)
BX = DX = extended memory above 16M, in 64K blocks

读了几次也可能不太明白描述的意思，因为缺少一些背景知识：RAM 在映射到 cpu 地址空间时，本应该是连续的，但由于某些原因，某段空间不能用来映射 RAM，这段空间就叫做 memory hole。在地址 1M - 16M 之间可能有一个 memory hole，后面的地址空间还可能有 memory hole，E801 最多只能处理 2 个 memory hole，也即最多返回 2 段连续的映射到 RAM 的地址空间，16M 之前和之后的。再回过头来看其英文描述，是不是感觉好理解了？或者简单一句话：E801 reports the amount of memory after 0xFFFFF

E801 的用法很简单，无需入参，直接 INT 0x15 AX = 0xE801，看代码前最好参考[这里](https://wiki.osdev.org/Detecting_Memory_(x86)#BIOS_Function:_INT_0x15.2C_AX_.3D_0xE801)的描述。看代码如何处理：

	static int detect_memory_e801(void)
	{
		...
		/* Do we really need to do this? */
		/* 有些 bios 只会返回ax/bx，或者 cx/dx */
		if (oreg.cx || oreg.dx) {
			oreg.ax = oreg.cx;
			oreg.bx = oreg.dx;
		}

		if (oreg.ax > 15*1024) {
			return -1;	/* Bogus! 因为 16M - 1M = 15M */
		} else if (oreg.ax == 15*1024) {
			/* 说明 1M -15M 之间没有 memory hole，又因为 bx 的值是 64k 为单位，
			 * 所以 << 6 转为单位 k*/
			boot_params.alt_mem_k = (oreg.bx << 6) + oreg.ax;
		} else {
			/*
			 * This ignores memory above 16MB if we have a memory
			 * hole there.  If someone actually finds a machine
			 * with a memory hole at 16MB and no support for
			 * 0E820h they should probably generate a fake e820
			 * map.
			 */
			/* 上面的注释讲的清楚，同时可参考：
			 * https://stackoverflow.com/questions/20612874/why-memory-size-returned-by-e801-bios-interrupt-is-ingnored-on-linux */
			boot_params.alt_mem_k = oreg.ax;
		}
	}

0x88 中断就更简单了，用于返回地址 0x100000 之后的连续空间的 size，以 k 为单位。

go_to_protected_mode 是 main 函数中的最后 & 最重要一步：

	void go_to_protected_mode(void)
	{
		/* Hook before leaving real mode, also disables interrupts */
		/* 进入 protect mode 前，需要关闭普通中断 & NMI。调用 realmode_swtch hook 完成
		 * 这项工作，如果有的话。同时引出一个小问题，为什么在切换到 protect mode 时需要关闭
		 * 中断？其中一个最简单的原因，real mode 下使用的是 IVT，protect mode 下使用的叫
		 * IDT，要做切换的话，可想而知要先关闭中断 */
		realmode_switch_hook();

		/* Enable the A20 gate。有趣的内容，下面详细分析 */
		if (enable_a20()) {
			puts("A20 gate not responding, unable to boot...\n");
			die();
		}

		/* Reset coprocessor (IGNNE#) */
		/* 各种 I/O port 操作 */
		reset_coprocessor();

		/* Mask all interrupts in the PIC */
		/* I/O 指令写入 OCW1 到 master & slave，设置 8259 PIC 的 IMR 寄存器，
		 * 屏蔽所有 IR pin*/
		mask_all_interrupts();

		/* Actual transition to protected mode... */
		/* 保护模式下需要使用 IDT，GDT，并将 table 的基址 load 到相应的寄存器。奇怪的是，
		 * 这里给 IDTR 的值是 0。显然这里使用的 table 都是临时的。protected_mode_jump
		 * 函数是定义在 pmjump.S 的汇编代码，下面详细分析。*/
		setup_idt();
		setup_gdt();
		protected_mode_jump(boot_params.hdr.code32_start,
				    (u32)&boot_params + (ds() << 4));
}

*enable_a20* 函数值得分析。首先了解什么是 a20，读完下面两篇就足够了解背景知识以及代码原理：

1. [A20 line@wikipedia](https://en.wikipedia.org/wiki/A20_line)
2. [A20 line@osdev](https://wiki.osdev.org/A20_Line)

最后参考一下 Intel SDM Volume 3a, chaper 8.7.13.4 的描述：
>On an IA-32 processor, the A20M# pin is typically provided for compatibility with the Intel 286
processor. Asserting this pin causes bit 20 of the physical address to be masked (forced to zero) for all external bus memory accesses.

>The functionality of A20M# is used primarily by older operating systems and not used by modern operating systems. On newer Intel 64 processors, A20M# may be absent.

来看代码：

	int enable_a20(void)
	{
		int loops = A20_ENABLE_LOOPS;
		int kbc_err;

		while (loops--) {
			/* First, check to see if A20 is already enabled(legacy free, etc.) */
			/* 此函数是重点，下面几个只是 enable a20 的不同方法。
			 * 原理：从 real mode 下表示相同线性地址的两个逻辑地址中(其中一个故意 wrap
			 * around)分别读取一个数，判断是否相等，相等则说明是从同一个地址读出来的，说明
			 * a20 没有 enable。
			 *
			 * 本例中，读取中断向量 0x80 所在逻辑地址 0：200h(线性地址 200h)处的 4-byte
			 * 整数值，++ 后写入原地址；从另一个逻辑地址 ffff:210h(real mode 下线性地址
			 * 也是 200h) 处读取它，判断是否相等，进而判断 a20 是否 enable。上述步骤最多
			 * 重复 A20_TEST_LONG 次，若不相等则说明 a20 已 enable，函数返回。
			 */
			if (a20_test_short())
				return 0;

			/* 下面几种 enable a20 的方法，在上面第 2 篇文章里都有描述，不是本文重点 */
			/* Next, try the BIOS (INT 0x15, AX=0x2401) */
			enable_a20_bios();
			if (a20_test_short())
				return 0;

			/* Try enabling A20 through the keyboard controller */
			kbc_err = empty_8042();

			if (a20_test_short())
				return 0; /* BIOS worked, but with delayed reaction */

			if (!kbc_err) {
				enable_a20_kbc();
				if (a20_test_long())
					return 0;
			}

			/* Finally, try enabling the "fast A20 gate" */
			enable_a20_fast();
			if (a20_test_long())
				return 0;
		}

		return -1;
	}

省略 I/O port 操作的两个函数的分析。setup 的代码中经常使用 in/out 指令来操作 I/O port，比如常见的 io_delay 函数，这里有一个[简单介绍](http://lkml.iu.edu/hypermail/linux/kernel/0802.2/0766.html)。

real mode 的 setup 代码最后一步是调用 protected_mode_jump:

	protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));

第一个入参 code32_start 表示 protect mode 内核代码所在地址，在 header.s 中定义为默认值 0x100000，grub 没有修改它，并且也是使用这个地址(GRUB_LINUX_BZIMAGE_ADDR)加载 protect mode 的代码; 第二个入参是全局变量 boot_param 的地址，以整数形式表示，表达式的意思是使用逻辑地址翻译成线性地址。这是汇编代码写的函数，定义在 arch/x86/boot/pmjump.S 中。这里有个小 tips, linux kernel real mode 的代码都使用了 GCC 的编译选项 `-mregparm=3`，定义在 arch/x86/Makefile 中：

	REALMODE_CFLAGS := $(M16_CFLAGS) -g -Os -DDISABLE_BRANCH_PROFILING \
	                   -Wall -Wstrict-prototypes -march=i386 -etregparm=3 \
	                   -fno-strict-aliasing -fomit-frame-pointer -fno-pic \
	                   -mno-mmx -mno-sse

`man gcc` 可知它的含义是：

>mregparm=num

>	Control how many registers are used to pass integer arguments.  By default, no registers are used to pass arguments, and at most 3 registers can be used.  You can control this behavior for a specific function by using the function attribute "regparm".

本来 x86 上函数调用时的参数传递是使用 stack 的，用这个选项则强制使用寄存器(最多3个)，可能是因为需要提高执行效率的原因。按入参从左到右的顺序，此选项分别使用 eax, edx, ecx 三个寄存器作参数传递用。

	/*
	 * void protected_mode_jump(u32 entrypoint, u32 bootparams);
	 */
	GLOBAL(protected_mode_jump)
		movl	%edx, %esi		# Pointer to boot_params table

		xorl	%ebx, %ebx # 清零 ebx
		movw	%cs, %bx
		shll	$4, %ebx # 获得 cs 的段基址，保存在 ebx
		addl	%ebx, 2f # 将 ebx 的值加到下方的 label: 2 处
		jmp	1f			# Short jump to serialize on 386/486
	1:

		# __BOOT_DS = 24，是 DS 的 segment selector value； __BOOT_TSS = 32，
		# 是 TSS 的 segment selector value。参考 segment selector 格式。
		movw	$__BOOT_DS, %cx
		movw	$__BOOT_TSS, %di

		movl	%cr0, %edx
		orb	$X86_CR0_PE, %dl	# Protected mode，这里就无需解释了
		movl	%edx, %cr0

		# Transition to 32-bit mode
		# 0x66 是 Operand-size override prefix，因为从 16 bit 代码跳到 32 bit 代码。
		# 0xea 是 jmp 指令，表示 Jump far, absolute, address given in operand
		.byte	0x66, 0xea # ljmpl opcode
	2:	.long	in_pm32	   # in_pm32 的值是它在当前 segment 中的 offset, 即 effective
						   # address。上面的代码将 real mode 下 cs 的段基址加到这里，
						   # 此时 label:2 处的这个数表示 real mode 下 in_pm32 的线性
						   # 地址；进入 protect mode 后所有 segment 的基址是 0，所以
						   # 这个数既是 real mode 下的线性地址，又是 protect mode
						   # 下的 offset(effective address)。

		.word	__BOOT_CS  # segment。GDT 中的段都以 0 为基地址，配上面的 offset
		                   # 一起作为跳转地址
	ENDPROC(protected_mode_jump)

		.code32
		.section ".text32","ax"
	GLOBAL(in_pm32)
		# Set up data segments for flat 32-bit mode
		# cx 寄存器在上面的 real mode 代码中已被赋值为数据段的 segment selector：__BOOT_DS
		movl	%ecx, %ds
		movl	%ecx, %es
		movl	%ecx, %fs
		movl	%ecx, %gs
		movl	%ecx, %ss

		# The 32-bit code sets up its own stack, but this way we do have
		# a valid stack if some debugging hack wants to use it.
		# ebx 在上面被赋值为 real mode 下 cs 的段基址，加到 esp 上，得到 esp 在 real
		# mode 下的线性地址。因为 protect mode 下所有段基址为 0，原来的线性地址变成现在的
		# offset(effective address)。所以，实际上还是使用原来的 stack。
		addl	%ebx, %esp

		# Set up TR to make Intel VT happy。实际没用
		ltr	%di

		# Clear registers to allow for future extensions to the
		# 32-bit boot protocol
		# 唯独 esi 没有清零，因为作为入参到目前还没使用。
		xorl	%ecx, %ecx
		xorl	%edx, %edx
		xorl	%ebx, %ebx
		xorl	%ebp, %ebp
		xorl	%edi, %edi

		# Set up LDTR to make Intel VT happy。同样，实际没用
		lldt	%cx

		# 本函数的第一个入参保留到现在才使用，又是一个 absolute jump
		jmpl	*%eax			# Jump to the 32-bit entrypoint
	ENDPROC(in_pm32)

上面代码中多次提到 make Intel VT happy，可以参考这个 [commit](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31) 了解一下。

linux kernel 的 real mode 代码终于结束，跳入了 protect mode。

### arch/x86/boot/vmlinux.bin

bzImage 中的另一部分: arch/x86/boot/vmlinux.bin，包含了压缩后的 kernel 本尊(源码根目录下的 vmlinux)和重定位信息。

需要先简单了解它是如何生成的：源码根目录下的 vmlinux 被 `objcopy -R .comment -S` 为 arch/x86/boot/compressed/vmlinux.bin; 若 kernel 编译为 relocatable 的，还将剥离其重定位信息到 vmlinux.relocs; 二者被一起压缩为 vmlinux.bin.gz(默认压缩算法)，作为同目录下 host program `mkpiggy` 的输入，生成 piggy.S; piggy.S 和同目录的其他源代码文件一起编译生成该目录下的 vmlinux；此 vmlinux 被 objcopy 处理输出为 arch/x86/boot/vmlinux.bin. 有 host program `build` 将 vmlinux.bin 与 setup.bin 一起打包成 bzImage。

图示的 bzImage 文件布局重点强调了 vmlinux.bin 的内容:

![Alt text](bzimagefilelayout.png)

图示的比例无法准确描述 bzImage 的内部 layout，从图示中的 size 数据(从我的环境中获得)可以看出，几乎整个 bzImage(vmlinux.bin) 的内容是 compressed kernel.

有必要简单了解几个术语先，因为下文将多处使用他们，在 "VO/ZO" 一节中有更多的细节描述。注意：表示同一个概念时，不同的地方可能使用不同的术语，混合使用容易让读者产生混淆，本文尽量阐明。

 1. VO: 指源码根目录下的 vmlinux, 从代码角度来说是 VO__end - VO__text 范围内的内容, 有时也被叫做 decompressed kernel；
 2. ZO: 上面的图示清楚的标记了它的范围，它的内容是 boot/compressed/vmlinux, 有时也被叫做 ZO image 或 decompressor(因为最重要的功能是解压缩 kernel);
 3. compressed kernel: 压缩的数据实际上包括 compressed/vmlinux.bin 和 compressed/vmlinux.relocs, 但通常简称为 compressed kernel.

以 x86-64 为例分析这部分代码。首先看下 arch/x86/boot/compressed/vmlinux 的代码布局，定义在 arch/x86/boot/compressed/vmlinux.lds：

	...
	SECTIONS
	{
	 . = 0;
	 .head.text : {
	  _head = . ;
	  *(.head.text)
	  _ehead = . ;
	 }

	 .rodata..compressed : {
	  *(.rodata..compressed)
	 }

	 .text : {
	  _text = .;
	  *(.text)
	  *(.text.*)
	  _etext = . ;
	 }
	 ....
	}

由此可知，地址 0 处的代码是 .head.text section 的内容，也即 real mode 的 setup 代码跳转到 protect mode 后开始执行的代码。.head.text section 在文件 arch/x86/boot/compressed/head_64.S 中。入口点是 startup_32。

#### startup_32

		__HEAD
		.code32
	ENTRY(startup_32)
		/*
		 * 32bit entry is 0 and it is ABI so immutable!
		 * If we come here directly from a bootloader,
		 * kernel(text+data+bss+brk) ramdisk, zero_page, command line
		 * all need to be under the 4G limit.
		 */
		/* clear direction flag in EFLAGS 寄存器，字符串操作指令(scas, stos等)将递增
		 * index 寄存器。下面初始化 page table 会用到 stos 指令。header.S 中已看到这条
		 * 指令，这里看到它是因为 32-bit boot protocol 的情况 */
		cld

		/* startup_32 是 32-bit boot protocol 的入口，在阅读下面代码前，有必要事先 &
		 * 事后各详细阅读一遍 “32-bit BOOT PROTOCOL” of Documentation/x86/boot.txt，
		 * 因为事先读肯定无法透彻理解，待看完代码再回头复习一遍便可查漏补缺。
		 *
		 * tips: bzImage 是作为一个整体被 load 到 RAM 中，所以在 32/64-bit boot
		 * protocol 下, boot loader 才能基于 setup 中 header.S 的 setup header
		 * 数据初始化 boot_params 供后续使用。
		 */

		/*
		 * Test KEEP_SEGMENTS flag to see if the bootloader is asking
		 * us to not reload segments
		 * setup 代码最后一个函数 protected_mode_jump 的入参之一是 setup 的数据结构
		 * boot_params 的线性地址，保存在 esi 中。以 grub 为例，它没有设置 KEEP_SEGMENTS
		 * bit，所以代码不会跳转到 1f，而是执行下面几行代码重新加载各 segment register。
		 * BP_loadflags 的实现待分析，但顾名思义，BP = Boot Protocol
		 */
		testb $KEEP_SEGMENTS, BP_loadflags(%esi)
		jnz 1f

		/* realmode_switch_hook 中已 cli，这是 32-bit boot protocol 的情况 */
		cli

		/* 如果是从 setup 来到 startup_32，则使用 setup 的 setup_gdt 函数配置的 GDT
		 * 加载段寄存器(pmjump.S 中有同样操作)；如果通过 32-bit boot protocol 来到
		 * startup_32，根据 “32-bit BOOT PROTOCOL” of Documentation/x86/boot.txt
		 * 的最后一段描述可知，GDT 已经按需配好。 但这里显式 load 段寄存器可能是为了保险。
		 */
		movl	$(__BOOT_DS), %eax
		movl	%eax, %ds
		movl	%eax, %es
		movl	%eax, %ss
	1:

	/*
	 * Calculate the delta between where we were compiled to run
	 * at and where we were actually loaded at.  This can only be done
	 * with a short local call on x86.  Nothing else will tell us what
	 * address we are running at.  The reserved chunk of the real-mode
	 * data at 0x1e4 (defined as a scratch field) are used as the stack
	 * for this calculation. Only 4 bytes are needed.
	 */
	/* ZO(protect mode kernel) 被加载到内存的地址，由 boot protocol(header.S) 中的
	 * code32_start(0x100000/1M) 表示, 即 boot loader 应将加载地址写入这个 field.
	 *
	 * 16-bit boot protocol 的 grub 代码直接使用 GRUB_LINUX_BZIMAGE_ADDR 作为加载
	 * ZO 的地址, 且 setup 中使用 code32_start 跳转到 protect mode kernel.
	 * 32-bit boot-protocol 时 grub 代码在不同情况下分别使用 GRUB_LINUX_BZIMAGE_ADDR
	 * 或字段 pref_address 作为 ZO 的加载地址。
	 *
	 * call 指令将下一条指令地址(label 1 的运行地址)压栈; 再弹出到 ebp，再和 label 1
	 * 的链接地址相减，差值放 ebp 中。这个差值(delta)即是符号 startup_32 的运行地址，
	 * 因为 startup_32 的链接地址是 0. (16-bit boot protocol 下它的运行地址是 0x100000)
	 * setup 中有设置 esp，这里再设置应该也是 32-bit boot protocol 的情况。
	 */
		leal	(BP_scratch+4)(%esi), %esp
		call	1f
	1:	popl	%ebp
		subl	$1b, %ebp

	/* setup a stack and make sure cpu supports long mode. */
	/* 本文件底部定义了 label: boot_stack，boot_stack_end，boot_heap，并开辟了空间 */
		movl	$boot_stack_end, %eax
		addl	%ebp, %eax /* stack 顶的编译地址加上 delta，得到运行时地址 */
		movl	%eax, %esp /* 得到栈的运行时地址，现在可以做函数调用(call)了 */

	/* 函数定义在 verify_cpu.S，文件在下面被 include。函数大部分内容是用 cpuid 指令
	 * discover CPU feature。此刻只需看 verify_cpu.S 的文件头描述，了解其返回值:
	 *     verify_cpu, returns the status of longmode and SSE in register %eax,
	 *     0: Success, 1: Failure
	 * 有需求时再来详细分析
	 */
		call	verify_cpu
		testl	%eax, %eax
		jnz	no_longmode  /* 根据常识，一般不会执行这条指令 */

	/*
	 * Compute the delta between where we were compiled to run at
	 * and where the code will actually run at.
	 *
	 * %ebp contains the address we are loaded at by the boot loader and %ebx
	 * contains the address where we should move the kernel image temporarily
	 * for safe in-place decompression.
	 */
	/* kernel_alignment 在 header.S(boot protocol) 中定义为: CONFIG_PHYSICAL_ALIGN,
	 * x86_64 下，其范围是 0x200000(2M) - 0x1000000(16M)，且必须是 2M 的倍数，可能因为
	 * early boot page table 使用 2M 的 page. 16-bit boot protocol 的 grub 没有
	 * 修改它. 重要 tip: CONFIG_PHYSICAL_ALIGN 的注释说必须是 2M(x86_64) 的倍数，但
	 * 实际上必须是 2,4,8,16 只一，所以其实原注释不够严谨，虽然没错，因为在 arch/x86/
	 * include/asm/boot.h 中有代码对它做检查：
	 *
	 *     #if (CONFIG_PHYSICAL_ALIGN & (CONFIG_PHYSICAL_ALIGN-1)) || \
	 *         (CONFIG_PHYSICAL_ALIGN < MIN_KERNEL_ALIGN)
	 *	       # error "Invalid value for CONFIG_PHYSICAL_ALIGN"
	 *	   #endif
	 *
	 * ebp 是 ZO 的加载地址(0x100000). CONFIG_RELOCATABLE 下，代码将该地址向上对齐到
	 * kernel_alignment，对齐后的值放 ebx. 比较 LOAD_PHYSICAL_ADDR 和 ebx，若
	 * ebx < LOAD_PHYSICAL_ADDR， 则给 ebx 赋值 LOAD_PHYSICAL_ADDR. 说明即使 ZO 是
	 * relocatable, 其最小加载地址也不能小于 LOAD_PHYSICAL_ADDR. arch/x86/Kconfig
	 * 中的符号 “RELOCATABLE” 有描述此行为。
	 *
	 * 所以 ebx 是解压缩 buffer 的地址，也即 VO 的起始地址？
	 * 2019/12/29 update：上述推论正确！ LOAD_PHYSICAL_ADDR 在代码中被定义为:
	 * CONFIG_PHYSICAL_START 对齐到 CONFIG_PHYSICAL_ALIGN。阅读 arch/x86/Kconfig
	 * 中 PHYSICAL_START 和 PHYSICAL_ALIGN 的定义，会发现上面的推论正确！！！
	 */
	#ifdef CONFIG_RELOCATABLE
		movl	%ebp, %ebx
		movl	BP_kernel_alignment(%esi), %eax
		decl	%eax
		addl	%eax, %ebx
		notl	%eax
		andl	%eax, %ebx
		/* 若 ebx >= LOAD_PHYSICAL_ADDR，则跳到 1f */
		cmpl	$LOAD_PHYSICAL_ADDR, %ebx
		jge	1f
	#endif
		movl	$LOAD_PHYSICAL_ADDR, %ebx

	/* Target address to relocate to for decompression */
	/* _end 定义在 linker script，表示 arch/x86/boot/compressed/vmlinux 的链接结束
	 * 地址，因链接起始地址是 0, 它也表示 ZO 的 memory image size, 比文件size 大，
	 * 因为 SHT_NOBITS 类型的 section(.bss, .pgtable) 不占据文件空间;
	 *
	 * init_size 在 header.S 中的定义比较复杂，在 “VO/ZO” 一节中有详细描述。它表示可
	 * 安全 in-place decompress 所需的 buffer size, 经实际计算发现, 一般情况下它等于
	 * ZO_INIT_SIZE;
	 *
	 * 二者相减的 offset 加到 ebx(解压缩 buffer 的地址)，所以，现在 ebx 表示 ZO memory
	 * image 被 copy/relocate 到解压缩 buffer 中的地址. 即 ZO 紧贴 buffer 的底部摆放.
	 * 压缩后 size 大于压缩前的情况是怎样???
	 */
	1:
		movl	BP_init_size(%esi), %eax
		subl	$_end, %eax
		addl	%eax, %ebx

	/* Prepare for entering 64 bit mode */

		/* Load new GDT with the 64bit segments using 32bit descriptor */
		/* GDT 定义在文件下方的 .data section. ebp 是 ZO 的加载/运行地址，加上它得到
		 * label: gdt 的运行地址。虽然 lgdt，但未重新 load 段寄存器，所以此刻没有生效.
		 */
		addl	%ebp, gdt+2(%ebp)
		lgdt	gdt(%ebp)

		/* Enable PAE mode。开启 long mode 必要步骤(见下文)之一，也是开启 4-level
		 * paging 的必要条件之一: CR0.PG=1 & CR4.PAE=1 & IA32_EFER.LME=1 */
		movl	%cr4, %eax
		orl	$X86_CR4_PAE, %eax
		movl	%eax, %cr4

	/* Build early 4G boot pagetable */

		/* If SEV is active then set the encryption mask in the page tables.
		 * This will insure that when the kernel is copied and decompressed
		 * it will be done so encrypted. */
		/* 此函数定义在 mem_encrypt.S。AMD Secure Encrypted Virtualization (SEV)
		 * 是 AMD 的特性，代码也是最近合入, 不是本文焦点，略过。直接跳到 1: 继续分析 */
		call	get_sev_encryption_bit
		xorl	%edx, %edx
		testl	%eax, %eax
		jz	1f
		subl	$32, %eax	/* Encryption bit is always above bit 31 */
		bts	%eax, %edx	/* Set encryption mask for page tables */

	1:
		/* Initialize Page tables to 0 */
		/*
		 * label: pgtable 定义在本文件最底部的 .pgtable section 中，指向 size 为
		 * BOOT_PGT_SIZE 的空间，它的地址在 arch/x86/boot/compressed/vmlinux.lds.S
		 * 中被对齐到 4k(page size)！ 注意！这里 leal 指令是基于 ebx 寄存器，上文说过，
		 * ebx 是 ZO memory image 被 relocate 到解压缩 buffer 中的地址，所以初始化
		 * 的页表位于解压缩 buffer 中 ZO 的相应空间。
		 *
		 * 阅读页表代码须掌握基础知识，权威材料是 Intel SDM 3a, "chapter 4: paging".
		 * 由上面注释可知，early boot 只需映射 4G 的物理地址空间，所以只需 6 个 paging
		 * structure，下面会详细解释为什么 4G 空间只需 6 个。*/
		leal	pgtable(%ebx), %edi
		xorl	%eax, %eax
		movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
		rep	stosl

		/* 5-level paging 出现前，x86_64 进入 long mode 时只可能是 4-level paging
		 * mode, 即: 48-bit linear address -> 52-bit(最大) physical address。
		 * 5-level paging 出现后，因无法预知 CPU 是否需要进入 5-level paging，所以仍
		 * 默认进入 4-level paging。
		 * linear address 的 48-bit 包含几个 fields，最后一个 field 表示页中的 offset，
		 * 其他都是 paging structure 的 index。
		 *   一个 PML4 entry 可以映射 2^39(48-9)，即 512G 物理地址空间；
		 *   一个 PDPT(page directory pointer table) entry 可以映射 2^30(48-9-9)，
		 *     即 1G 物理地址空间；
		 *   一个 PD(page directory) entry 可以映射 2^21(48-9-9-9)，即 2M 地址空间。
		 * 所以，映射 4G 物理地址空间需要 PML4 的 1 个 entry，1 个 PDPT 中的 4 个 entry，
		 * 4 个 PD 中的全部 entry 即可。另外，paging structures 的 size 都是 4k，CR3
		 * 保存 top paging structure 的物理地址，由 Intel SDM 3a, chapter 2.5
		 * CONTROL REGISTERS 可知，CR3 只保存 lower 12 bits 以外的其他地址 bit，
		 * 地址的 lower 12 bits 是 0, 所以第一级 paging structure 的地址是 4k 对齐的；
		 * 同样，其他所有 paging structure 的地址都是 4k 对齐的；若 entry 是 map a
		 * page，则 entry 中的地址是 page size 对齐的。 本例中，所有 paging structure
		 * 分配在同一块连续内存，所以他们的地址都是 4k 对齐的。
		 */

		/* Build Level 4 */
		/* 这里的 0x1007 让我困惑了一天，主要因为忽略了 lable: pgtable 已经 4k 对齐。
		 * 将 (pgtable + 0x1007) 的 effective address 放到 eax 中，作为 PML4 第一个
		 * entry 的内容。举例：假设 pgtable = 0x80600000，则 0x80600000 + 0x1007 =
		 * 0x80601007，作为 entry 的值，说明此 entry 指向的下一级 paging structure 的
		 * 地址是 0x80601000；末尾的 7 = 01111b，表示 Present；相应的 memory region
		 * 是 read/write;user-mode address(参考 4.6.1 Determination of Access
		 * Rights)。所以，这算是小小的 trick。
		 * 如上面分析所说，PML4 table 仅需初始化 1 个 entry
		 */
		leal	pgtable + 0(%ebx), %edi
		leal	0x1007 (%edi), %eax
		movl	%eax, 0(%edi)
		addl	%edx, 4(%edi) /* 由 AMD SEV 特性引入，可忽略。同样忽略下面所有相同操作 */

		/* Build Level 3 */
		/* 如上面分析，PDPT 需要初始化 4 个 entry */
		leal	pgtable + 0x1000(%ebx), %edi
		leal	0x1007(%edi), %eax
		movl	$4, %ecx /* counter，用来 count 4 个 entry */
	1:	movl	%eax, 0x00(%edi)
		addl	%edx, 0x04(%edi) /* AMD SEV 特有，忽略 */
		addl	$0x00001000, %eax
		addl	$8, %edi /* entry size 是 8 byte */
		decl	%ecx
		jnz	1b

		/* Build Level 2 */
		/* 如上面分析，PD table 的所有 entry 都需初始化，共 4 tables x 512 entries
		 * per table。0x183 = 0001,1000,0011b，参考 Intel SDM 3a, Figure 4-11.
		 * Formats of CR3 and Paging-Structure Entries with 4-Level Paging 可知：
		 * Present, Read/Write, Page size = 1(2M page), Global。第一个 page 的
		 * 地址是 0，所以是从物理地址 0 开始连续 map 4G 物理地址空间
		 */
		leal	pgtable + 0x2000(%ebx), %edi
		movl	$0x00000183, %eax
		movl	$2048, %ecx /* 4 x 512 = 2048 */
	1:	movl	%eax, 0(%edi)
		addl	%edx, 4(%edi) /* AMD SEV 特有，忽略 */
		addl	$0x00200000, %eax /* page size 是 2M */
		addl	$8, %edi
		decl	%ecx
		jnz	1b

		/* Enable the boot page tables */
		/* 获得解压缩 buffer 中 top paging structure 运行(物理)地址。此刻还没开启
		 * paging. 因 ZO 链接起始地址是 0，label 的值表示地址，也表示 offset. Tip: lea
		 * 得到的 effective address 是运行地址。问题：和 CS 基址有没有联系？
		 */
		leal	pgtable(%ebx), %eax
		movl	%eax, %cr3

		/* Enable Long mode in EFER (Extended Feature Enable Register) */
		/* 参考 Intel SDM 3a, "2.2.1 Extended Feature Enable Register".
		 * MSR 的一般性介绍在 Intel SDM 3a, “9.4 MODEL-SPECIFIC REGISTERS (MSRS)”;
		 * 详细介绍在 Intel SDM 4, chapter 2.
		 *
		 * 开启 long mode 的条件：protect mode(CR0.PE=1) 下，LME=1 & CR0.PG=1。
		 * 已经在 protect mode 下，这里 LME=1 只是 enable，需要 CR0.PG=1 才会
		 * activate。这也表示，long mode 中必须有 paging 功能。
		 * 详细描述在 Intel SDM 3a, 9.8.5 Initializing IA-32e Mode
		 */
		movl	$MSR_EFER, %ecx
		rdmsr		/* 64-bit MSR 地址由 ECX 指定，读入 EDX:EAX */
		btsl	$_EFER_LME, %eax /* enable long mode in value of eax，然后写回 msr */
		wrmsr

		/* After gdt is loaded */
		/* 为了 make Intel VT happy，在 real mode 的 setup 代码有做过，上文有描述。
		 * 这是 32-bit boot protocol 的情况。segment selector 的格式参考
		 * Intel SDM 3a, Figure 3-6. Segment Selector */
		xorl	%eax, %eax
		lldt	%ax
		movl    $__BOOT_TSS, %eax
		ltr	%ax

		/*
		 * Setup for the jump to 64bit mode
		 *
		 * When the jump is performed we will be in long mode but
		 * in 32bit compatibility mode with EFER.LME = 1, CS.L = 0, CS.D = 1
		 * (and in turn EFER.LMA = 1).	To jump into 64bit mode we use
		 * the new gdt/idt that has __KERNEL_CS with CS.L = 1.
		 * We place all of the values on our mini stack so lret can
		 * used to perform that far jump.
		 */
		/* Intel 对于 64-bit 模式的权威描述是：Intel 64-bit 架构引入了 IA-32e mode，
		 * 它包括 2 种子模式, Compatibility mode 和 64-bit mode。compatibility mode
		 * 下执行 16-bit 或 32-bit 代码, 64-bit mode 下执行 64-bit 代码。通过
		 * EFER.LME = 1 进入 long mode(即 IA-32e mode) 时，势必是子模式的一种，究竟是
		 * 哪儿一种，由 CS.L 决定。代码的意图是直接跳入 64-bit mode，所以需要 load
		 * 64-bit code segment: __KERNEL_CS.这里使用了小技巧：push segment
		 * selector 和 startup_64 的地址，待执行 lret 指令将它们加载回 CS:EIP。
		 * 问题: 不能用长跳转 jmp cs:eip 的形式跳转到另一个代码段中执行吗？ AT&T 语法：
		 * ljmp $SECTION,$OFFSET。(2019/12/29)经测试, ljmp $__KERNEL_CS, %eax 报错：
		 *     Error: operand type mismatch for `ljmp'
		 * ljmp $__KERNEL_CS, %rax 则报错：Error: bad register name `%rax'
		 * 粗略分析：根据 volume 2 指令集手册中的 JMP 描述，jmp 要么跟 ptr16:32, 两部分
		 * 都是立即数；要么 m16:64，两部分是存在 memory 中。所以写法不对。
		 */
		pushl	$__KERNEL_CS
		leal	startup_64(%ebp), %eax
	#ifdef CONFIG_EFI_MIXED
		/* 若 firmware 是 EFI，推测 efi_32_config 处已被填入数据，那么 eax 将被更新为
		 * 别的地址。本文暂不考虑 EFI 的情况，所以直接跳到 1f */
		movl	efi32_config(%ebp), %ebx
		cmp	$0, %ebx
		jz	1f
		leal	handover_entry(%ebp), %eax
	1:
	#endif
		pushl	%eax

		/* Enter paged protected Mode, activating Long Mode */
		/* 问：startup_32 中的代码明显已经处在 protect mode，为什么再 enable 一次？
		 * 答：(2019/12/29)经测试，删掉 X86_CR0_PE 导致无法启动，又回到 grub menu。
		 * WHY? 仔细看宏定义发现，这两个 macro 只有 set 自己的 bit。这里操作不是读取了
		 * CR0 后仅 enable PG bit，然后写回。所以如果删除 X86_CR0_PE，它的 bit 最终
		 * 不会被 set。从另一个角度看，CR0 中有那么多 bit，这里不管之前有没有设置其他 bit，
		 * 单纯的只 set PG 和 PE，似乎也透露出什么信息
		 */
		movl	$(X86_CR0_PG | X86_CR0_PE), %eax /* Enable Paging and Protected mode */
		movl	%eax, %cr0

		/* 看起来 compatibility mode 的存在也是为了润滑的过渡到 64-bit mode。protect
		 * mode 下，所有 long mode 必需开关开启后，直接进入 compatibility mode。差别
		 * 仅在 CS.L，切换个 CS，即可从 compatibility mode 进入 64-bit mode。
		 *
		 * 从 stack 上 load 回准备好的 CS 和 EIP，进入 64-bit mode. */
		/* Jump from 32bit compatibility mode into 64bit mode. */
		lret
	ENDPROC(startup_32)

startup_32 的主要作用是为跳入到 long mode 做准备，Intel SDM 3a, 9.8.5 Initializing IA-32e Mode 中描述了所需步骤:

>1. 处在没有开启 paging 的 protect mode 下。
>2. 设置 CR4.PAE = 1 使能 physical-address extensions (PAE)
>3. 加载 paging structure 的物理地址到 CR3 寄存器。64-bit 下即 Level 4 page map table (PML4) 的基址
>4. 设置 IA32_EFER.LME = 1 使能 long mode(IA-32e)
>5. 设置 CR0.PG = 1 使能 paging。这将使 processor 自动设置 IA32_EFER.LMA = 1.

32-bit protect mode 时 paging 不是必须开启，从上述步骤可以看出，IA-32e mode 默认必须开启 paging。对照手册中的步骤回望代码，会发现代码做的和手册讲的完全一致。手册对第 5 步还有更多描述：

>The MOV CR0 instruction that enables paging and the following instructions must be located in an **identity-mapped** page

Tips:

>1. processor 操作模式切换条件在 Intel SDM 3a, Figure 2-3. Transitions Among the Processor’s Operating Modes
>2. paging mode 切换条件在 Intel SDM 3a, Figure 4-1. Enabling and Changing Paging Modes

#### 科普两例

代码中常看到 **identity-mapped** 字眼，是什么意思呢？术语解释在 [Identity function](https://en.wikipedia.org/wiki/Identity_function)，原来是个数学术语，中文翻译为恒等函数。单词 identity 的主要意思是**身份**，但还有另一个意思：**一致性**，所以翻译过来是“恒等映射的 page”，也就是 linear address = physical address 的映射。

另一科普：看到过很久却没有找到权威定义的术语: Canonical Address，权威解释在 Intel SDM 1, 3.3.7.1 Canonical Addressing。

#### startup_64

继续分析 head_64.S:

	/* 忽略 efi32_stub_entry 的分析。因为我们不考虑 firmware 是 EFI 的情况 */

	/* startup_64 是 64-bit boot protocol 的入口。同样，在阅读下面代码前，有必要事先 &
	 * 事后各仔细阅读一遍 “64-bit BOOT PROTOCOL” of Documentation/x86/boot.txt。
	 * grub 不支持 64-bit boot protocol, EFI 等新的 firmware 才支持 */

		.code64
		.org 0x200  /* WHY？(2019/1/2) 这是 64-bit boot protocol 的约定 */
	ENTRY(startup_64)
		/*
		 * 64bit entry is 0x200 and it is ABI so immutable!
		 * We come here either from startup_32 or directly from a
		 * 64bit bootloader.
		 * If we come here from a bootloader, kernel(text+data+bss+brk),
		 * ramdisk, zero_page, command line could be above 4G.
		 * We depend on an identity mapped page table being provided
		 * that maps our entire kernel(text+data+bss+brk), zero page
		 * and command line.
		 */

		/* Setup data segments. */
		/* 64-bit mode 下，根据 Intel SDM volume 1, chapter 3.4.2.1 Segment
		 * Registers in 64-Bit Mode 所说：不管 CS, DS, ES, SS 中的值如何，他们的
		 * 段基址都被当作 0；FS, GS 例外，这俩的使用如往常一样。所以这里给各段寄存器赋值
		 * 的意义是？而且他们都被赋值为0，引用 null descriptor？
		 * By initializing the segment registers with segment selector of null
		 * descriptor, accidental reference to unused segment registers can be
		 * guaranteed to generate an exception.
		 * 这是目前 manual 中看到的最像答案的说法, BUT 应该并不是！Still in Question!
		 * 无论是从 startup_32 还是 64-bit boot protocol 来到这里，所有段寄存器应该是
		 * 设置过了，所以暂不理解为什么还要显式的设置一遍？
		 * 2019/1/15 update: 由 commit 08da5a2ca 可知，set fs/gs 是为了 make VT
		 * happy, 但 BIOS 中开启了 Intel VT, 没有发现异常！
		 * 2019/2/12 update: 询问社区(https://lkml.org/lkml/2019/1/15/443)后依然
		 * 不明白. PIC 需要所有 segment register 为 0? 但测试过删除这几行代码，并没有
		 * 问题，从 startup_32 走到这里时，一些 segment register 明显不为 0. TBD.
		 */
		xorl	%eax, %eax
		movl	%eax, %ds
		movl	%eax, %es
		movl	%eax, %ss
		movl	%eax, %fs
		movl	%eax, %gs

		/* 这段 & 更下面的代码的逻辑在 startup_32 中出现过，同样是因为 64-bit boot
		 * protocol 的原因又做一遍，下面的英文注释也有讲。 重复一遍 FYI:
		 * 将 ZO 的加载物理地址，向上对齐到 kernel_alignment 后放入 ebp，作为解压缩
		 * buffer 的起始地址; ebx 是 ZO memory image 被 copy/relocate 到解压缩
		 * buffer 中的物理地址。
		 *
		 * 2019/2/12 update: 起初以为下面 comments 中的 "2M boundary" 不严谨，认为是
		 * aligned to CONFIG_PHYSICAL_ALIGN, 但看过 CONFIG_PHYSICAL_ALIGN 的注释，
		 * 悟了一下后，发现下面的 comments 没有问题，因为 CONFIG_PHYSICAL_ALIGN 必须是
		 * 2M 的倍数，即可以是: 2,4,6,8,10,12,14,16, 举例来说，如果数字 N 是 6 的倍数，
		 * 那么肯定也是 2 的倍数，a subtle difference ~ ~!
		 */

		/*
		 * Compute the decompressed kernel start address.  It is where
		 * we were loaded at aligned to a 2M boundary. %rbp contains the
		 * decompressed kernel start address.
		 *
		 * If it is a relocatable kernel then decompress and run the kernel
		 * from load address aligned to 2MB addr, otherwise decompress and
		 * run the kernel from LOAD_PHYSICAL_ADDR
		 *
		 * We cannot rely on the calculation done in 32-bit mode, since we
		 * may have been invoked via the 64-bit entry point.
		 */

		/* Start with the delta to where the kernel will run at. */
	#ifdef CONFIG_RELOCATABLE
		/* 由于不仔细阅读 gnu as 文档对 rip relative addressing 的描述，仍然按 x86_32
		 * memory reference 方式来理解这条 lea 指令，导致无法理解，以致费了一天多时间翻阅
		 * Intel SDM 也没有找到对该语句的权威解释。绝望中重翻 `info as`，finally got:
		 *
		 * AT&T: 'symbol(%rip)', Intel: '[rip + symbol]'
		 *     Points to the 'symbol' in RIP relative way, this is shorter than
		 *     the default absolute addressing.*/
		 * (In 9.15.7 Memory References of `info as`)
		 *
		 * 原来就是 symbol 的地址！而且是运行时地址！ So, everything finally make sense
		 * 看下这条语句的反汇编：
		 *
		 *   20c:  48 8d 2d ed fd ff ff  lea  -0x213(%rip),%rbp  # 0 <startup_32>
		 * (诚不欺我)
		 */
		leaq	startup_32(%rip) /* - $startup_32 */, %rbp
		movl	BP_kernel_alignment(%rsi), %eax
		decl	%eax
		addq	%rax, %rbp
		notq	%rax
		andq	%rax, %rbp
		cmpq	$LOAD_PHYSICAL_ADDR, %rbp
		jge	1f
	#endif
		movq	$LOAD_PHYSICAL_ADDR, %rbp

		1:
		/* Target address to relocate to for decompression */
		/* rbp 是解压缩 buffer 的物理起始地址. 计算后，rbx 是 ZO memory image 被 copy/
		 * relocate  到解压缩 buffer 中物理地址 */
		movl	BP_init_size(%rsi), %ebx
		subl	$_end, %ebx
		addq	%rbp, %rbx

		/* Set up the stack. 使用 relocated 后的 ZO 中的 boot_stack_end 做 stack */
		leaq	boot_stack_end(%rbx), %rsp

		/*
		 * paging_prepare() and cleanup_trampoline() below can have GOT
		 * references. Adjust the table with address we are running at.
		 *
		 * Zero RAX for adjust_got: the GOT was not adjusted before;
		 * there's no adjustment to undo.
		 */
		/* 上面注释和 adjust_got 的注释连在一起看更 make sense，但不理解位置无关代码的话，
		 * 对注释的理解只能停留在表面。关于动态链接，共享对象，位置无关代码(PIC)，对我来说目前
		 * 最 make sense 的讲解是：<一个程序员的自我修养>，第7章“动态链接”。
		 *
		 * 为何 ZO 中有 Global Offset Table(GOT)? 分析：使用 -fPIC 或 -fPIE 编译位置
		 * 无关代码，会生成 .got section，因为 GOT 是实现 PIC 的核心技术。
		 *
		 * 查看引入上面注释的 commit 5c9b0b1c498 可知：为生成 X86_64 的位置无关代码，新
		 * 版本 binutils 使用 RIP relative addressing 的方式(这也是我脑中最自然的方式)，
		 * 而老版本 binutils 则使用 GOT。
		 *
		 * 下面的分析是个人粗浅理解，很可能不准确，仅作理解当前代码用。盼指正。
		 * 本例中，GOT entry 的内容应是相应符号的编译地址，因不涉及动态链接，不会 involve
		 * Procedure Link Table(PLT)。entry size 在 i386 下为 4 bytes，X86_64 下
		 * 为 8 bytes。普通可执行程序与动态库链接时默认使用 -fPIC，对每一个动态库中符号的
		 * 引用都生成一条 GOT entry；程序执行时，run time linker 会修改 GOT entry 中的
		 * 值，使其等于目标符号的运行时地址。可执行程序主体中所有对 .so 中符号的引用都变成对
		 * 自身 GOT 中相应 entry 的引用。
		 *
		 * 对老版本 binutils 会生成 GOT 不完全理解，因为 ZO image 作为独立运行的个体，不引用
		 * 外部模块的变量或函数，所以不存在动态链接。是部分函数如 paging_prepare，
		 * cleanup_trampoline 还是全部函数都会在 GOT 中有 reference? 推测: 对全部
		 * global C 函数都会生成 GOT entry, head_32.S 中也有调整 got 的相同操作。
		 *
		 * 暂时不纠结这个问题，因为老版本中 -fPIC 会生成 GOT 是事实，那么只有 bootloader
		 * 有机会对它做 PIC 的调整(对于普通应用程序来说，叫动态链接)，但从未听说有此说法，且
		 * Zero RAX for adjust_got 的注释和代码，arch/x86/boot/compressed/Makefile
		 * 中 "quiet_cmd_check_data_rel" 的注释，也证实 bootloader 不会对 compressed
		 * kernel 做 PIC 的调整，这个调整就是对 GOT entry 的内容重新计算。既然之前没有人做
		 * GOT adjust，所以此刻 adjust_got 无需 undo previous adjustment，只需 apply
		 * current adjustment。   那么又引出别的问题：adjustment 的值如何决定？GOT 中的
		 * 初始值是什么？这两个问题本质上是一个，假设并推理一下：GOT 的内容是符号的编译地址，
		 * 所以此处 adjust_got 只需加上编译地址和加载地址之 delta，即可更新为运行地址，又
		 * 因为编译起始地址是 0，所以加载地址就是这个 delta。
		 *
		 * 我的环境中，binutils 的版本是 2.29(>2.24)，所以不会通过 GOT 实现位置无关，通过
		 * `objdump -h <compressed_vmlinux>` 发现，.got section 的 size 是 0x18，即
		 * 只有 3 条(8 bytes per entry) entry，而 GOT 的前三条是特殊 entry，用于动态链接，
		 * 后面的 entry 才是对符号的 reference，也就是说，这是只有 header 的 GOT，其实是
		 * 空的，所以不会使用它。
		 * ToDo: 虚拟机装个老版本 binutils 看一下 compressed 目录下的 vmlinux？
		 */
		xorq	%rax, %rax

		/*
		 * Calculate the address the binary is loaded at and use it as
		 * a GOT adjustment.
		 */
		call	1f
	1:	popq	%rdi
		subq	$1b, %rdi

		/* 定义在文件下方 */
		call	adjust_got

		/*
		 * At this point we are in long mode with 4-level paging enabled,
		 * but we might want to enable 5-level paging or vice versa.
		 *
		 * The problem is that we cannot do it directly. Setting or clearing
		 * CR4.LA57 in long mode would trigger #GP. So we need to switch off
		 * long mode and paging first.
		 *
		 * We also need a trampoline in lower memory to switch over from
		 * 4- to 5-level paging for cases when the bootloader puts the kernel
		 * above 4G, but didn't enable 5-level paging for us.
		 *
		 * The same trampoline can be used to switch from 5- to 4-level paging
		 * mode, like when starting 4-level paging kernel via kexec() when
		 * original kernel worked in 5-level paging mode.
		 *
		 * For the trampoline, we need the top page table to reside in lower
		 * memory as we don't have a way to load 64-bit values into CR3 in
		 * 32-bit mode.
		 *
		 * We go though the trampoline even if we don't have to: if we're
		 * already in a desired paging mode. This way the trampoline code gets
		 * tested on every boot.
		 */
		/* 使用文件下方自己定义的 GDT。5-level paging 的背景知识参考：
		 *  <5-Level Paging and 5-Level EPT white paper>。总结上面注释的要点如下：
		 *   1. 无法在 long mode 中随意切换 4-level 和 5-level paging，只能从 protect
		 *      mode 直接跳入，所以我们需要一个在低地址中的 trampoline 函数
		 *   2. trampoline 函数的功能不仅那么点，还有很多其他的功能。
		 *      目前作者还没有感受，待补充。
		 */
		/* Make sure we have GDT with 32-bit code segment */
		/* 64-bit boot protocol 的情况，如 commit 7beebaccd50 所说，bootloader 可能
		 * 没有为我们准备一个合适的 GDT，即缺少 32-bit code segment，因为 trampoline
		 * 必须位于低地址(0-4G)，这是 32-bit protect mode 的虚拟地址空间范围
		 */
		leaq	gdt(%rip), %rax
		movq	%rax, gdt64+2(%rip)
		lgdt	gdt64(%rip)

		/*
		 * paging_prepare() sets up the trampoline and checks if we need to
		 * enable 5-level paging.
		 *
		 * Address of the trampoline is returned in RAX.
		 * Non zero RDX on return means we need to enable 5-level paging.
		 *
		 * RSI holds real mode data and needs to be preserved across
		 * this function call.
		 */
		/* 对 paging_prepare 的使用解释的很清楚，函数细节分析在下方。这里涉及 X86_64 ABI
		 * 中的 calling conventions，详细描述在 “3.2.3 Parameter Passing” of
		 * https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf
		 * 简单总结：函数入参传递要么使用 register 要么使用 stack。规则是：先对所有入参
		 * 分类，分为 POINTER, INTEGER, MEMORY 等等，查看 paging_prepare 函数发现其
		 * 入参是指针(POINTER)，根据规则：
		 *   If the class is INTEGER or POINTER, the next available register of
		 *   the sequence %rdi, %rsi, %rdx, %rcx, %r8 and %r9 is used
		 * 这就是为什么代码需要 push & mov rsi。
		 *
		 * 返回值和入参一样分类，对于 structure/array 这种 aggregate 类型，按照规则：
		 *   If the size of the aggregate exceeds a single eightbyte, each is
		 *   classified separately. Each eightbyte gets initialized to class
		 *   NO_CLASS.
		 * 每一个 field 单独分类，在根据规则得到一个 resulting class，本例适用规则：
		 *   If both classes are equal, this is the resulting class.
		 * 函数返回值是结构体，其中两个 field 都是 long 型整数。所以 resulting class 是
		 * INTEGER，根据规则:
		 *   If the class is INTEGER or POINTER, the next available register of
		 *   the sequence %rax, %rdx is used.
		 * 所以现在可以理解上面的原注释了。
		 * brief introduction 见 Figure 3.4: Register Usage
		 */
		pushq	%rsi
		movq	%rsi, %rdi		/* real mode address */
		call	paging_prepare
		popq	%rsi

		/* Save the trampoline address in RCX */
		movq	%rax, %rcx

		/*
		 * Load the address of trampoline_return() into RDI.
		 * It will be used by the trampoline to return to the main code.
		 */
		/* lea 指令拿到符号的运行地址，为了后面的 absolute jump */
		leaq	trampoline_return(%rip), %rdi

		/* Switch to compatibility mode (CS.L = 0 CS.D = 1) via far return */
		/* 跳入 trampoline 函数: trampoline_32bit_src. rax 是 trampoline 空间的
		 * 起始地址，lea 拿到 trampoline 函数在其中的地址。
		 * lret again, why not ljmp? 答：与 startup_32 中的 lret 是同样的问题，
		 * 在上面已解释。
		 */
		pushq	$__KERNEL32_CS
		leaq	TRAMPOLINE_32BIT_CODE_OFFSET(%rax), %rax
		pushq	%rax
		lretq

	trampoline_return:
		/* Restore the stack, the 32-bit trampoline uses its own stack */
		/* 回忆：rbx 是 ZO memory image 被 copy/relocate 到解压缩 buffer 中的物理地址.
		 * boot_stack_end 的值这里用作 offset, 以 buffer 中 ZO 的空间作 stack */
		leaq	boot_stack_end(%rbx), %rsp

		/*
		 * cleanup_trampoline() would restore trampoline memory.
		 *
		 * RDI is address of the page table to use instead of page table
		 * in trampoline memory (if required).
		 *
		 * RSI holds real mode data and needs to be preserved across
		 * this function call.
		 */
		/* cleanup_trampoline 的入参是指针，根据 calling conventions，用 rdi 传递。
		 * top_pgtable 定义在本文件最末尾。很久不看此段代码的话，容易和 startup_32 中
		 * 初始化页表用的 pgtable 混淆。top_pgtable 是专用于存储 trampoline 空间中的
		 * page table。函数内容本身很简单：把 trampoline 空间的 top page table 放到
		 * 它的专用空间，恢复 trampoline 空间原来的内容。下面有更详细分析。
		 */
		pushq	%rsi
		leaq	top_pgtable(%rbx), %rdi
		call	cleanup_trampoline
		popq	%rsi

		/* 截至目前，被 relocate 到 buffer 的 ZO memory image 空间在代码中只有两处使用:
		 * 1. 作 stack；2. 存放页表, 即刚刚 copy 的 top_pgtable，及 startup_32 中
		 * 初始化页表 pgtable。恰好都是 .bss section 的空间。
		 */

		/* Zero EFLAGS。为什么需要？ */
		pushq	$0
		popfq

		/*
		 * Previously we've adjusted the GOT with address the binary was
		 * loaded at. Now we need to re-adjust for relocation address.
		 *
		 * Calculate the address the binary is loaded at, so that we can
		 * undo the previous GOT adjustment. 解释很清楚，无需赘述。
		 */
		call	1f
	1:	popq	%rax
		subq	$1b, %rax

		/* The new adjustment is the relocation address */
		/* 温馨提示： rbx 是 ZO memory image 被 copy/relocate 到解压缩 buffer 中的
		 * 物理地址，用作新的 GOT adjustment。在当前运行地址处使用 relocated 后的地址
		 * 做完 PIC 调整，就可将 ZO memory image 拷贝到 relocated 处 & 跳过去执行了
		 */
		movq	%rbx, %rdi
		call	adjust_got

	/*
	 * Copy the compressed kernel to the end of our buffer
	 * where decompression in place becomes safe.
	 */
	/* _bss 定义在 linker script，_bss 之前的 section 是待 copy 的内容。
	 * ZO 的编译起始地址是 0，所以 _bss 的值也表示待 copy 的 size. std(Set Direction
	 * Flag) 指令表示使用逆序 copy，即从内容的后端->前端进行 copy; 当前运行在 long mode,
	 * 所以每次可 copy 一个 quadruple word. 要注意! 虽整体是逆序 copy, 但 copy 一个单位
	 * (8 bytes)时是正常的顺序! 否则 copy 后内容就乱了，这就是 _bss-8 的含义! So，copy
	 * 次数自然是 size/8，因 .bss 地址(待 copy size)在 linker script 中已对齐到
	 * 64(2^6), 所以 size/8 的结果肯定是整数。
	 * copy 结束则 cld 恢复 direction flag. rsi 保存着 boot_params 的地址，字符串
	 * 操作要用到它，所以 push 保存一下。
	 */
		pushq	%rsi
		leaq	(_bss-8)(%rip), %rsi  /* 源地址 */
		leaq	(_bss-8)(%rbx), %rdi  /* 目的地址 */
		movq	$_bss /* - $startup_32 */, %rcx  /* copy 的 size */
		shrq	$3, %rcx  /* copy 的次数 = size/8*/
		std
		rep	movsq
		cld
		popq	%rsi

	/* Jump to the relocated address. */
	/* 再提醒: ZO 编译起始地址是 0，所以 label 既是地址，又是 offset. lea 出 relocated
	 * ZO 中 label： relocated 的 effective 地址，then, absolute jump to it.
	 * 自此，将从解压缩 buffer(relocated 地址) 的 label relocated: 处执行，也就是说，
	 * 后面运行的代码是 ZO 在 relocated 处的拷贝。
	 */
		leaq	relocated(%rbx), %rax
		jmp	*%rax

	#ifdef CONFIG_EFI_STUB
		/* 不关注 EFI 相关流程，snip... */
	#endif

	/* 重要细节！！！解压后下方 Jump to the decompressed kernel 指令不会被覆盖的原因！*/
		.text
	relocated:

	/*
	 * Clear BSS (stack is currently empty)
	 * 与上面 copy 的逻辑相似，但不是逆序，且 _ebss 的值已 aligned to 8 bytes(_bss
	 * aligned to 64 bytes). 原来 stack 和 heap 被放在了 .bss section，且已做完所有
	 * 的函数调用，所以原注释强调现在 stack is empty.
	 */
		xorl	%eax, %eax
		leaq    _bss(%rip), %rdi
		leaq    _ebss(%rip), %rcx
		subq	%rdi, %rcx  /* _ebss - _bss, 获得 size */
		shrq	$3, %rcx    /* 一次 copy quadruple word(8 bytes) */
		rep	stosq

	/*
	 * Do the extraction, and jump to the new kernel..
	 * extract_kernel 有 6 个入参，都是指针或整数，返回指针。根据 calling conventions，
	 * 从左到右，入参依次使用 %rdi, %rsi, %rdx, %rcx, %r8 and %r9；返回值通过 %rax。
	 * 因 extract_kernel 内容复杂，涉及到 kaslr 处理，elf 解析，重定位处理，将在下文
	 * 作为独立一节进行分析。
	 */
		pushq	%rsi			/* Save the real mode argument */
		movq	%rsi, %rdi		/* real mode address */
		leaq	boot_heap(%rip), %rsi	/* malloc area for uncompression */
		leaq	input_data(%rip), %rdx  /* input_data */
		movl	$z_input_len, %ecx	/* input_len. */
		movq	%rbp, %r8		/* output target address。看C函数后可知，作物理地址 */
		movq	$z_output_len, %r9	/* decompressed length, end of relocs */
		call	extract_kernel		/* returns kernel location in %rax */
		popq	%rsi

	/*
	 * Jump to the decompressed kernel.
	 * (背景乐响起：我们走进新时代！)跳转！Finally！本文件的任务结束！
	 */
		jmp	*%rax

	/*
	 * Adjust the global offset table
	 *
	 * RAX is the previous adjustment of the table to undo (use 0 if it's the
	 * first time we touch GOT).
	 * RDI is the new adjustment to apply.
	 */
	/* jae 的 ae = above or equal，很少看到 above 的跳转条件，和 great 的区别是啥？参考：
	 * https://stackoverflow.com/questions/20906639/difference-between-ja-and-jg-in-assembly
	 * https://en.wikibooks.org/wiki/X86_Assembly/Control_Flow#Jump_if_Above_(unsigned_comparison)
	 *
	 * 符号 _got 和 _egot 定义在 linker script 中，又是通过 RIP relative addressing，
	 * 获得两个符号的运行时地址，放在 rdx, rcx 中。
	 * 代码本身比较容易理解：通过比较 .got section 起始 & 结束地址，遍历 GOT 的 entry，
	 * undo 之前的 adjustment，apply 现在的 adjustment。
	 * GOT 前三项是特殊 entry，所以这里需要从第一项开始 adjust 吗，从第4项不行吗？
	 * `readelf -x .got vmlinux` 显示确实如此，除了第一项是 .dynamic section 的地址，
	 * 其他两项都是空(0)，猜测 .dynamic section 未来应该也不会被用到，而且被 objcopy 为
	 * 上层目录的 vmlinux.bin 时(objcopy --strip-all)删除了所有符号信息 & 重定位信息.
	 * 2019/1/2：还不了解打包到 bzImage 的 relocation info 是什么情况，那些被 strip
	 * 的 relocation info 说不定在 bzImage 中。
	 */
	adjust_got:
		/* Walk through the GOT adding the address to the entries */
		leaq	_got(%rip), %rdx
		leaq	_egot(%rip), %rcx
	1:
		cmpq	%rcx, %rdx
		jae	2f  /* 起始地址怎么会大于结束地址？所以这里不会跳到 2f，但下面的代码将对起始地址++ */
		subq	%rax, (%rdx)	/* Undo previous adjustment */
		addq	%rdi, (%rdx)	/* Apply the new adjustment */
		addq	$8, %rdx	/* 起始地址不断++，直到等于结束地址，跳转到 2f，返回 */
		jmp	1b
	2:
		ret

	.code32
	/*
	 * This is the 32-bit trampoline that will be copied over to low memory.
	 *
	 * RDI contains the return address (might be above 4G).
	 * ECX contains the base address of the trampoline memory.
	 * Non zero RDX on return means we need to enable 5-level paging.
	 */
	/* 上面的注释 so sweet~ */
	ENTRY(trampoline_32bit_src)
		/* Set up data and stack segments */
		/* 因为 startup_64 入口处用 0 初始化了 cs 外所有 data segments，此刻我们运行在
		 * compatible mode，所以需要重新初始化数据段 */
		movl	$__KERNEL_DS, %eax
		movl	%eax, %ds
		movl	%eax, %ss

		/* Set up new stack。因为下面要 push 操作，所以需要准备 stack。用 trampoline
		 * 空间的结束地址作栈顶，因为 trampline 函数很小，4kb 中还剩余很多。 */
		leal	TRAMPOLINE_32BIT_STACK_END(%ecx), %esp

		/* Disable paging。终究要重新回到 long mode，所以先关闭 paging */
		/* btr: Bit Test and Reset. stores the value of the bit in the CF flag,
		 *      and clears the selected bit in the bit string to 0 */
		movl	%cr0, %eax
		btrl	$X86_CR0_PG_BIT, %eax
		movl	%eax, %cr0

		/* Check what paging mode we want to be in after the trampoline */
		/* rdx(edx) 是 l5_required，表示是否需要进入 5-level paging mode. 原注释很精准 */
		cmpl	$0, %edx
		jz	1f /* edx 为 0，不需开启 5-level paging，意味着：4->4 或者 5->4 */

		/* We want 5-level paging: don't touch CR3 if it already points to 5-level page tables */
		/* 需要进入 5-level paging mode, 有两种情况： 4->5 或 5->5. 通过 CR4.LA57 bit
		 * 判断进入 trampoline 函数前是否在 5-level paging mode 下。*/
		movl	%cr4, %eax
		testl	$X86_CR4_LA57, %eax
		jnz	3f  /* 5->5 的情况，不需要动 CR3，仅回到 long mode 即可 */
		jmp	2f  /* 4->5 的情况，跳到 2: 处理 5- 所需页表 */
	1:
		/* We want 4-level paging: don't touch CR3 if it already points to 4-level page tables */
		/* 希望进入 4-level paging mode, 检查进入 trampoline 函数前是否在 5-level
		 * paging mode 下，这涉及是否使用 trampoline 空间的页表。通过 CR4.LA57 bit
		 * 来确认 */
		movl	%cr4, %eax
		testl	$X86_CR4_LA57, %eax
		jz	3f /* 之前不在 5-level paging mode 下，就是 4->4, 跳到 3: */
	2:
		/* Point CR3 to the trampoline's new top level page table */
		/* 5->4 或 4->5 的情况，所以使用 trampoline 中准备好的 page table. */
		leal	TRAMPOLINE_32BIT_PGTABLE_OFFSET(%ecx), %eax
		movl	%eax, %cr3
	3:
		/* Enable PAE and LA57 (if required) paging modes */
		/* label 3 的功能是打开 paging mode 所需开关，与页表操作的代码解耦。各 sub-开关
		 * 先打开，等待最后 activate paging via CR0.PG */
		movl	$X86_CR4_PAE, %eax  /* 不 care CR4 中其他 bit? */
		cmpl	$0, %edx
		jz	1f /* 确认不需进入 5-level paging */
		orl	$X86_CR4_LA57, %eax /* 按需 eanble 5-level paging bit */
	1:
		movl	%eax, %cr4 /* enable PAE & LA57(需要的话), 但还没 activate */

		/* trampoline 核心代码看完了，感叹其几个 label 使用之精妙，主要就是 2: 和 3:，
		 * 前者只负责把 top paging structure 的地址搬到 cr3，后者只负责 enable 所需
		 * 开关；第一个 1: 之前是进入 5-level 的处理，之后是进入 4-level 的处理。下面的
		 * 代码是共用的功能：activate & 返回 long mode
		 */

		/* Calculate address of paging_enabled() once we are executing in the trampoline */
		/* 获得 trampoline 函数的 exit 地址，准备离开 trampoline. */
		leal	paging_enabled - trampoline_32bit_src + TRAMPOLINE_32BIT_CODE_OFFSET(%ecx), %eax

		/* Prepare the stack for far return to Long Mode */
		pushl	$__KERNEL_CS
		pushl	%eax

		/* Enable paging again。为啥需要 PE 一起呢，它不是已 enable？答：这个问题在
		 * 上文已解释：这两个宏只 set 各自的 bit，只使用 X86_CR0_PG 的话，意味着 clear
		 * 了 PE bit
		 */
		movl	$(X86_CR0_PG | X86_CR0_PE), %eax
		movl	%eax, %cr0

		lret /* 回到 long mode，跳到 trampoline 空间的 paging_enabled: */

		.code64
	paging_enabled:
		/* Return from the trampoline。出现多次的 absolute jump */
		jmp	*%rdi

		/*
	     * The trampoline code has a size limit.
	     * Make sure we fail to compile if the trampoline code grows
	     * beyond TRAMPOLINE_32BIT_CODE_SIZE bytes.
		 */
		/* 终于解释 TRAMPOLINE_32BIT_CODE_SIZE 了，paging_prepare() 使用它做 copy */
		.org	trampoline_32bit_src + TRAMPOLINE_32BIT_CODE_SIZE

		.code32
	no_longmode:
		/* This isn't an x86-64 CPU, so hang intentionally, we cannot continue */
	1:
		hlt
		jmp     1b

	#include "../../kernel/verify_cpu.S"

		.data
	/* gdt64 的四行是 commit 7beebaccd50 引入。表示 X86_64 下 GDTR 的内容, base 是 64
	 * bits，但分别 .long，.word，.quad 是何意? 经咨询: https://lkml.org/lkml/2019/1/22/420
	 * 应该是作者手误。遂投 patch(https://lkml.org/lkml/2019/1/23/198) 修正.
	 */
	gdt64:
		.word	gdt_end - gdt
		.long	0
		.word	0
		.quad   0
	gdt:
		/* 巧妙利用 GDT 中第一项是 null descriptor 的定义，用这个空间存储 GDTR 的值。
		 * 紧挨着的是 GDT 的内容
		 */
		.word	gdt_end - gdt   /* limit */
		.long	gdt				/* base address */
		.word	0				/* padding，补足 null descriptor */
		/* 参考 Intel SDM 3a，“3.4.5 Segment Descriptors”。前三项都是：base: 0x0,
		 * limit: 0xfffff; type 'a' 表示代码段，Execute/Read, type '2' 表示数据段，R/W;
		 * 9 = 1001b，表示 Segment present，不是 system descriptor, DPL=00; 左数第三个
		 * 数字 a = 1010b，表示 64-bit code segment，即此 segment 包含 native 64-bit
		 * code，也即运行在 IA-32e 的 64-bit mode 下, granularity 是 4k(若 L bit set，
		 * 则 D bit 必须 clear)； c 表示 granularity 是 4k，对于 32-bit code or data
		 * segment, D bit 总是 set 1. */
		/* 64-bit 模式下，TSS Descriptor 被扩展到 16 bytes。除此，还有 LDT Descriptor
		 * 等也是被扩展到 16 bytes。参考 Intel SDM 3a, 3.5.2 Segment Descriptor Tables
		 * in IA-32e Mode。引出个问题：64-bit mode 下如何计算 descriptor 的地址？对于
		 * 32-bit mode，手册有说：index x 8 + GDT base address。但 64-bit 下，
		 * descriptor 的 size varies
		 */
		.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
		.quad	0x00af9a000000ffff	/* __KERNEL_CS */
		.quad	0x00cf92000000ffff	/* __KERNEL_DS */
		.quad	0x0080890000000000	/* TS descriptor */
		.quad   0x0000000000000000	/* TS continued */
	gdt_end:

pgtable_64.c 的函数分析(以他们的出现顺序排列，所以和文件中的顺序不同)：

	/* 进入此函数时，CPU 在 long mode 下，但 paging mode 可能是 4-level，也可能是
	 * 5-level，要根据情况判断是否需要 paging mode 切换 */
	struct paging_config paging_prepare(void *rmode)
	{
		struct paging_config paging_config = {};

		/* Initialize boot_params. Required for cmdline_find_option_bool(). */
		/* 和 setup 中的 boot_params 同名，定义在 misc.c，需要通过它找 cmd_line_ptr，
		 * 下方的 cmdline_find_option_bool 函数会用到。但这个解析函数中的：
		 *     	cptr = cmdline_ptr & 0xf;
		 * 不明白，待向社区询问。2019/1/16 update: 不需问社区了，已明白，在下方命令行解析
		 * 一节有详细解释。
		 */
		boot_params = rmode;

		/*
		 * Check if LA57 is desired and supported.
		 *
		 * There are several parts to the check:
		 *   - if the kernel supports 5-level paging: CONFIG_X86_5LEVEL=y
		 *   - if user asked to disable 5-level paging: no5lvl in cmdline
		 *   - if the machine supports 5-level paging:
		 *     + CPUID leaf 7 is supported
		 *     + the leaf has the feature bit set
		 *
		 * That's substitute for boot_cpu_has() in early boot code.
		 */
		/* 判断 kernel 是否需要运行在 5-level paging. 如何判断，上面注释解释的很清楚。
		 * IS_ENABLED 宏定义在 kconfig.h 中，使用了一些技巧判断某配置项是否被定义，核心
		 * 技术是 variadic macro:
		 *     https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html
		 * tip: 调用有参数的宏时，即使不传入参，也可使用，参数为空而已。例子：
		 *     #include <stdio.h>
		 *     #define world(arg) "hello"##arg
		 *     int main()
		 * 	   {
		 *         printf("%s\n", world());
		 *	       return 0;
		 *     }
		 */
		if (IS_ENABLED(CONFIG_X86_5LEVEL) &&
				!cmdline_find_option_bool("no5lvl") &&
				native_cpuid_eax(0) >= 7 &&
				(native_cpuid_ecx(7) & (1 << (X86_FEATURE_LA57 & 31)))) {
			paging_config.l5_required = 1;
		}

		/* 无论是否切换 paging mode，head_64.S 中说 trampoline 函数始终需要(假设我们
		 * 同意)，所以先找到合适的 trampoline 空间：在 conventional memory(0 - 640kb)
		 * 中找到 2 pages 大小的空间，起始地址是 page(4k) 对齐的。
		 * 下有 find_trampoline_placement 的详细分析。
		 */
		paging_config.trampoline_start = find_trampoline_placement();
		/* 将刚找到的地址由整形数据变成 pointer */
		trampoline_32bit = (unsigned long *)paging_config.trampoline_start;

		/* Preserve trampoline memory. 占用的 trampoline 空间不知道有什么重要数据，
		 * 所以需要备份，待用完恢复 */
		memcpy(trampoline_save, trampoline_32bit, TRAMPOLINE_32BIT_SIZE);

		/* Clear trampoline memory first */
		memset(trampoline_32bit, 0, TRAMPOLINE_32BIT_SIZE);

		/* Copy trampoline code in place */
		/* trampoline 空间有两个作用：1. 存放 trampoline 函数的 copy; 2. 暂存 top
		 * level page table，用于 4->5 或 5->4 的 paging mode switch.
		 * 代码分析至此，一直未看到 TRAMPOLINE_32BIT_CODE_SIZE 的定义，原来它定义在
		 * head_64.S 中。但不理解为什么起始地址要除 sizeof(unsigned long)？
		 * 2019/01/11 update: 终于想明白了 = =|trampoline_32bit 的类型是
		 * (unsigned long *)，它加 1 的时候，实际是加了 sizeof(unsigned long)
		 */
		memcpy(trampoline_32bit + TRAMPOLINE_32BIT_CODE_OFFSET / sizeof(unsigned long),
				&trampoline_32bit_src, TRAMPOLINE_32BIT_CODE_SIZE);

		/*
		 * The code below prepares page table in trampoline memory.
		 *
		 * The new page table will be used by trampoline code for switching
		 * from 4- to 5-level paging or vice versa.
		 *
		 * If switching is not required, the page table is unused: trampoline
		 * code wouldn't touch CR3.
		 */

		/* 如需 paging mode 切换，则需准备好目标 mode 的页表，待执行 trampoline 函数时
		 * load 入 CR3：
		 *   1. 4 -> 5，少了最高层 paging structure，则在 trampoline 空间准备它；
		 *   2. 5 -> 4，多了最高层 paging structure，则将 PML4 copy 到 trampoline
		 *      空间, 当作新的 top level paging structure.
		 * mode 切换一共有 2 x 2 = 4 种情况: (事先 4- 或 5-) x (事后 4- 或 5-)。目前
		 * 推测，由 bootloader 启动 kernel 的情况下，只可能处于 4-level paging mode，
		 * 因为 firmware 没有需求进入 5-level
		 */

		/*
		 * We are not going to use the page table in trampoline memory if we
		 * are already in the desired paging mode.
		 */
		/* 此 if 语句 cover 事先事后相同的两种情况，这样则不需要准备页表 */
		if (paging_config.l5_required == !!(native_read_cr4() & X86_CR4_LA57))
			goto out;

		/* 就剩下两种情况： 4 -> 5 或 5 -> 4 */
		if (paging_config.l5_required) {
			/*
			 * For 4- to 5-level paging transition, set up current CR3 as
			 * the first and the only entry in a new top-level page table.
			 */
			/* 对于 identity mapping，这样确实就够了。entry 的多少决定映射 的 size，
			 * entry 的位置决定 linear address 到 physical address 的映射关系 */
			trampoline_32bit[TRAMPOLINE_32BIT_PGTABLE_OFFSET] = __native_read_cr3() | _PAGE_TABLE_NOENC;
		} else {
			unsigned long src;

			/*
			 * For 5- to 4-level paging transition, copy page table pointed
			 * by first entry in the current top-level page table as our
			 * new top-level page table.
			 *
			 * We cannot just point to the page table from trampoline as it
			 * may be above 4G.
			 */
			/* 将 5-level paging 的 2nd top paging table 拷贝到 trampoline 空间
			 * 作为 4-level paging 的 top paging table，所以只需 1 PAGE_SIZE 空间。
			 * 这隐含了前提：之前 5-level paging 是 identity mapping；且，内核
			 * 加载地址 + setup_header.init_size，zero page，command line buffer
			 * 都在第一个 PML5 entry 映射的范围内，即 512 x 512G = 256TB 空间内。
			 * 如何确认？ 这里隐含了一个前提：5->4 发生在 kexec 的情况下，kexec 加载第二
			 * 个内核时，会建立一个 identity mapped page table, 毕竟 kexec 可以看作
			 * 一个特殊的 boot loader. 从代码可以找到证据: machine_kexec_prepare ->
			 * init_pgtable, 可以猜想，kexec 建立的页表会满足此刻的所有需求。
			 * 
			 * __native_read_cr3() 本就是返回 unsigned long，这里转换的逻辑是：
			 * 以整数形式读出 top page table 的地址，转为指针后再使用 dereference
			 * operator (*) 可以取该地址处相应指针类型的整数，即 PML5 中第一个 entry
			 * 的内容，逻辑与 PAGE_MASK 就得到了第一个 PML4 table 的地址。
			 */
			src = *(unsigned long *)__native_read_cr3() & PAGE_MASK;
			memcpy(trampoline_32bit + TRAMPOLINE_32BIT_PGTABLE_OFFSET / sizeof(unsigned long),
			       (void *)src, PAGE_SIZE);
		}

	out:
		return paging_config;
	}

	/* 给 trampoline 函数找合适的空间摆放。看起来是在第 1M 空间內，为啥呢？推测：第 1M 内
	 * 是 BIOS、部分 bootloader(grub boot.img) 的运行空间，走入 long mode linux 后，
	 * 已不在需要他们(BIOS 的中断服务？)，而后面的空间是未来使用的，所以在第 1M 空间内找个空比较合理。
	 */
	/* 此函数参照 reserve_bios_regions() 实现，需阅读理解原函数注释，但仅阅读注释恐怕也
	 * 无法完全理解，还需要一些古老的背景知识：现代 PC 架构源自1981年的 IBM PC，BIOS 的
	 * 概念也源自该产品。IBM PC 只有 1M 地址空间，除了映射 RAM，还有显卡，ROM 中的 BIOS
	 * 等。参考：
	 *     https://en.wikipedia.org/wiki/IBM_Personal_Computer (随意阅读，干货不多)
	 * 这又引出 Conventional memory 的概念，参考：
	 *     https://en.wikipedia.org/wiki/Conventional_memory
	 *     https://ancientelectronics.wordpress.com/tag/conventional-memory
	 * Simply speaking: real mode 只能寻址 1M 地址空间，其中只有前 640kb 映射 RAM，
	 * 这 640kb 叫 conventional memory；后面 384kb(也叫 upper memory area)作它用，
	 * 比如映射显卡，ROM BIOS 等。BIOS 的数据会放在 Conventional memory 中，比如第 1k
	 * 中的 Interrupt Vector Table(IVT), 1k 到 1k + 256byte 的 BIOS Data Area(BDA),
	 * 紧贴 conventional memory 顶端放置的 Extended BIOS Data Area(EBDA)。
	 * 因 EBDA 被放在 conventional memory 的顶部，且 EBDA 在未来仍可能被用到，这会减少
	 * conventional memory 的 reported amount(INT 0x12)。参考：
	 *     1. http://www.ctyme.com/intr/rb-0598.htm (INT 0x12)
	 *     2. https://wiki.osdev.org/Memory_Map (很好！必看！)
	 *     3. http://stanislavs.org/helppc/bios_data_area.html (最后一节展示了
	 *        640k - 1M 空间映射的例子)
	 * 链接 2. 中说：SMM also seems to use the EBDA. So the EBDA memory area
	 * should never be overwritten. 所以函数的计算中，考虑到了这一点。
	 */
	static unsigned long find_trampoline_placement(void)
	{
		unsigned long bios_start, ebda_start;
		unsigned long trampoline_start;
		struct boot_e820_entry *entry;
		int i;

		/*
		 * Find a suitable spot for the trampoline.
		 * This code is based on reserve_bios_regions().
		 */

		 * 总的意思：在第 1M 地址空间里找一段合适的空间放 trampoline 函数，对这块空间有
		 * 什么要求？首先类型(当然)必须是 E820_TYPE_RAM，那就是 conventional memory；
		 * 其次不能碰 conventional memory 中 BIOS 相关的区域(代码，数据)，因为不知道
		 * 未来谁还会用到他们。
		 * 第 1M 中，除 conventional memory 映射为 RAM，后面的地址空间会映射为 BIOS
		 * firmare(由变量bios_start表示) 及显卡等，这部分空间不在 RAM 中。本函数尽可能
		 * 保守的推测可用的 RAM 空间，这就是为什么对齐时使用 round_down，而不是 round_up。
		 */
		/* 两个 magic number： 0x40e，0x413 需要解释。这两个值表示线性地址，由链接：
		 *     http://stanislavs.org/helppc/bios_data_area.html
		 *     https://wiki.osdev.org/EBDA#Overview
		 * 可知，BIOS data area(BDA) 中的逻辑地址 40:0E 处的 word 保存着 EBDA 的
		 * segment base address；逻辑地址 40:13 处的 word 保存着 Memory size in
		 * Kbytes(kilobytes of contiguous memory starting at absolute address
		 * 00000h，或叫 conventional memory size，但减去 EBDA 的空间)。变量名终于
		 * make sense：conventional memory 以上，1M 以下的部分是 BIOS related area，
		 * 所以叫 bios_start。但某种角度看，这两个值其实也是一个东西？
		 */
		ebda_start = *(unsigned short *)0x40e << 4;
		bios_start = *(unsigned short *)0x413 << 10;

		/* MIN(128K) 不知道如何确定的，MAX 是 636K，但注释中写 640K，这 4k 是 quirk?
		 * 考虑 Dell 可能在 RAM size 中没有预留 EBDA 的空间？ 通过 git blame ebda.c
		 * 发现引入 ebda.c 的 commit: 0c51a965ed3c4, 证明推测正确。但现在似乎不再需要
		 * 这 4k quirk？因为下面有 if(xxx && ebda_start < bios_start)，待确认
		 */
		if (bios_start < BIOS_START_MIN || bios_start > BIOS_START_MAX)
			bios_start = BIOS_START_MAX;

		/* conventional memory 中的 EBDA 是 BIOS related memory，不可以 overwritten，
		 * 理论上，正常情况下，我认为 ebda_start 应该等于 bios_start，但正因为一些奇葩
		 * 情况，如 Dell 老系统中 "RAM size" value 没有考虑 EBDA，会导致 bios_start
		 * 大于 ebda_start，这种情况当然要按实际情况来，即 EBDA memory 也要被 reserve.
		 * 因为原函数 reserve_bios_regions 的终极目标是，bios_start 至 1M 之间都要被
		 * reserve.
		 */
		if (ebda_start > BIOS_START_MIN && ebda_start < bios_start)
			bios_start = ebda_start;

		/* 可能是因为怕下面 for 循环中找不到合适的，而 bios_start 又需要始终保持页对齐，
		 * 所以先做页对齐。round up 可能就跑到 BIOS related area 中了，所以宁愿浪费
		 * 一点也要 round down。
		 */
		bios_start = round_down(bios_start, PAGE_SIZE);

		/* Find the first usable memory region under bios_start. */
		/* 0 - bios_start 是 conventional memory，bios_start 以上至 1M 的空间是被
		 * reserve 的，不能被 kernel 用作 available RAM。注意!这里是从低地址往上找
		 * 1st usable memory region under bios_start
		 */
		for (i = boot_params->e820_entries - 1; i >= 0; i--) {
			entry = &boot_params->e820_table[i];

			/* Skip all entries above bios_start. */
			if (bios_start <= entry->addr)
				continue;

			/* Skip non-RAM entries. */
			if (entry->type != E820_TYPE_RAM)
				continue;

			/* Adjust bios_start to the end of the entry if needed. */
			if (bios_start > entry->addr + entry->size)
				bios_start = entry->addr + entry->size;

			/* Keep bios_start page-aligned. */
			bios_start = round_down(bios_start, PAGE_SIZE);

			/* Skip the entry if it's too small. */
			if (bios_start - TRAMPOLINE_32BIT_SIZE < entry->addr)
				continue;

			break;
		}

		/* Place the trampoline just below the end of low memory */
		return bios_start - TRAMPOLINE_32BIT_SIZE;
	}

	void cleanup_trampoline(void *pgtable)
	{
		void *trampoline_pgtable;

		/* 除以 sizeof(unsigned long) 上文中已有解释 */
		trampoline_pgtable = trampoline_32bit + TRAMPOLINE_32BIT_PGTABLE_OFFSET / sizeof(unsigned long);

		/*
		 * Move the top level page table out of trampoline memory,
		 * if it's there.
		 */
		/* 如果 trampoline 函数做过 4->5 或 5->4 的 paging mode 切换，则此时 CR3 的值
		 * 是 trampoline 空间中 page table 的地址, 它是 top page table, 要把它放到
		 * 到预备好的 top page table 空间: pgtable 处. 看起来在 paging 开启的状态下，
		 * 可以随意修改 CR3 的值！
		 */
		if ((void *)__native_read_cr3() == trampoline_pgtable) {
			memcpy(pgtable, trampoline_pgtable, PAGE_SIZE);
			native_write_cr3((unsigned long)pgtable);
		}

		/* Restore trampoline memory。之前备份的内容恢复到原地址 */
		memcpy(trampoline_32bit, trampoline_save, TRAMPOLINE_32BIT_SIZE);
	}

#### command line parsing under x86/boot

此节的存在，来自分析 paging_prepare->cmdline_find_option_bool 时遇到的小困惑。command line 解析在 boot/ 和 compressed/ 目录下都会用到，在 boot/ 目录遇到时并没细看，但来到 compressed/ 后，就有了一些小困惑：fs/gs 寄存器的使用问题，本节旨在解释 fs/gs 使用在 boot/ 和 compressed/ 下的差异，并非 parsing 本身。So, let's go~

command line parsing 在 compressed/ 中第一次出现如上所述：paging_prepare->cmdline_find_option_bool("no5lvl")。compressed/cmdline.c 中 #include 了 boot/ 下的 cmdline.c，所以实际使用的核心函数来自 boot/cmdline.c，而 compressed/cmdline.c 中只是定义了 wrapper 函数和 helper 函数(set_fs, rdfs8)。来看 boot/cmdline.c 中的核心函数 __cmdline_find_option_bool：

	/* 仅关注 fs/gs 相关代码 */
	int __cmdline_find_option_bool(unsigned long cmdline_ptr, const char *option)
	{
		addr_t cptr;
		...

		/* 此函数既运行在 real mode, 也运行在 protect mode/long mode，两种情况分别分析。
		 * 入参 cmdline_ptr 是 command line 在 RAM 中的线性(物理)地址。
		 *
		 * 1. 作为 setup 的代码运行在 real mode
		 * 线性(即物理)地址的计算方式：linear address = segment base << 4 + offset,
		 * 第一行开始没想明白，后来逆向思考明白了，这两行代码后，按照 real mode 计算地址
		 * 的方式，fs << 4 + cptr = cmdline_ptr
		 *
		 * 2. 作为 ZO 运行在 protect mode/long mode
		 * 这个情况略复杂，因为地址计算方式不一样，此时的所有 segment base 都是 0。所以
		 * compressed/cmdline.c 定义了自己的 set_fs 和 rdfs8，代替 boot/ 中的。看
		 * 定义，便明白了。
		 */
		cptr = cmdline_ptr & 0xf;
		set_fs(cmdline_ptr >> 4);

		/* real mode 下，segment 的 size 是固定的 64k = 0x10000 */
		while (cptr < 0x10000) {
			c = rdfs8(cptr++);
			...
		}

		return 0;
	}

	/* compressed/cmdline.c */
	static unsigned long fs;
	static inline void set_fs(unsigned long seg)
	{
		fs = seg << 4;  /* shift it back */
	}

	typedef unsigned long addr_t;
	static inline char rdfs8(addr_t addr)
	{
		return *((char *)(fs + addr));
	}

#### extract_kernel:

如上文所述，extract_kernel 函数的内容复杂，涉及 kaslr 处理，elf 解析，重定位处理，本节将逐个进行分析。

函数定义在 arch/x86/boot/compressed/misc.c:

	/* asmlinkage，字面意思是汇编链接，在 X86_64 下定义为空；在 X86_32 下被定义为：
	 *     #define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
	 * 意为不使用 register 做参数传递，X86_32 最多可用 3 个 register 作入参传递。但这样做
	 * 的目的是什么？推测：如名字所示，汇编链接时用，即汇编代码调用 C 函数时用，为什么呢？因为
	 * C 语言函数之间的调用，有编译器会处理 calling conventions，而汇编代码则必须自己处理。
	 * 若用 register 做入参传递，X86_32 下第 3 个后的入参时只能 stack，混合使用 register
	 * 和 stack 传递入参的话，汇编代码处理起来有些复杂。
	 * 从另一个角度看，C 函数间调用的 calling conventions 有编译器根据指定的 rule 去 take
	 * care，但汇编代码没有这种机制，需要 coder 纯手动处理，那么，设一个明确的规则，也方便汇编
	 * 代码按照一致的方式处理入参。
	 * 参考 head_32.S 种调用 extract_kernel 时入参传递的汇编代码(用 stack，从左往右)。
	 * X86_64 本身支持最多 6 个 registers 做入参传递，所以定义为空。
	 * __visible 是另一个有趣的话题，它因内核链接优化而引入。背景知识参考：
	 * 		https://lwn.net/Articles/512548/
	 * 它的定义是：
	 * 		# define __visible   __attribute__((__externally_visible__))
	 * 参考 GCC 的属性：__externally_visible__，编译选项 ‘-flto’，‘-fwhole-program’
	 */
	asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
					  unsigned char *input_data,
					  unsigned long input_len,
					  unsigned char *output,
					  unsigned long output_len)
	{
		/* VO 的 memory image size, 包括了 .bss, .brk section */
		const unsigned long kernel_total_size = VO__end - VO__text;
		/* VO 的 virtual address 其实是 base on __START_KERNEL_map 的，但这里没有提 */
		unsigned long virt_addr = LOAD_PHYSICAL_ADDR;

		/* Retain x86 boot parameters pointer passed from startup_32/64. */
		boot_params = rmode;

		/* Clear flags intended for solely in-kernel use. */
		/* 没看到之前的代码有 set 此 flag, so, WHY? 推测可能其他的 bootloader 会 set
		 * 此 flag，总之以防万一。*/
		boot_params->hdr.loadflags &= ~KASLR_FLAG;

		/* 下面有大片代码都对本文重点不重要，比如 console debug output 之类，暂时略过 */
		...
		/* real mode 的 setup 代码调用过，此处是非 16-bit boot protocol 的情况 */
		console_init();
		debug_putstr("early console in extract_kernel\n");

		free_mem_ptr     = heap;	/* Heap */
		free_mem_end_ptr = heap + BOOT_HEAP_SIZE;

		/* Report initial kernel position details. */
		...

		/*
		 * The memory hole needed for the kernel is the larger of either
		 * the entire decompressed kernel plus relocation table, or the
		 * entire decompressed kernel plus .bss and .brk sections.
		 */
		/* KASLR 是本节重点只一，下方详细分析此函数。
		 * 倒数第二个入参的 max() 的意思是，压缩数据可能包括 relocation 信息，如果包括，
		 * 则 output_len 比 kernel_total_size 大; output 和 virt_addr 分别表示
		 * kaslr 后的物理地址和虚拟地址，作 out param 用。所以 output 的处理有点技巧：
		 * 传入时二重指针转为一重指针，函数定义是一重指针，因为就是要修改指针(地址)变量的值。
		 * 无 KASLR，无 relocation xxx...
		 */
		choose_random_location((unsigned long)input_data, input_len,
					(unsigned long *)&output,
					max(output_len, kernel_total_size),
					&virt_addr);

		/* Validate memory location choices. */
		/* 对选择的随机物理地址 & 虚拟地址做对齐验证。X86_64 下，MIN_KERNEL_ALIGN 是 2M*/
		if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
			error("Destination physical address inappropriately aligned");
		if (virt_addr & (MIN_KERNEL_ALIGN - 1))
			error("Destination virtual address inappropriately aligned");

	#ifdef CONFIG_X86_64
		if (heap > 0x3fffffffffffUL)
			error("Destination address too large");
		if (virt_addr + max(output_len, kernel_total_size) > KERNEL_IMAGE_SIZE)
			error("Destination virtual address is beyond the kernel mapping area");
	#else
		if (heap > ((-__PAGE_OFFSET-(128<<20)-1) & 0x7fffffff))
			error("Destination address too large");
	#endif

	#ifndef CONFIG_RELOCATABLE
		/* 无 CONFIG_RELOCATABLE 时，是解压缩 buffer 的地址(也即 VO 的运行物理地址)是
		 * LOAD_PHYSICAL_ADDR，在 head_64.S 中 enforced. 如果是非 relocatable 的
		 * ZO
		 * Tip: CONFIG_RELOCATABLE 仅影响解压 buffer 的地址，即 VO 的物理地址。
		 */
		if ((unsigned long)output != LOAD_PHYSICAL_ADDR)
			error("Destination address does not match LOAD_PHYSICAL_ADDR");
		if (virt_addr != LOAD_PHYSICAL_ADDR)
			error("Destination virtual address changed when not relocatable");
	#endif

		/* 核心解压缩函数，解压缩 kernel 到指定的物理地址 */
		debug_putstr("\nDecompressing Linux... ");
		__decompress(input_data, input_len, NULL, NULL, output, output_len,
				NULL, error);

		/* decompressor 对 VO 做个 validation, 比如：确认是它是 ELF 格式， */
		parse_elf(output);

		/* 依赖 CONFIG_X86_NEED_RELOCS. 因为虚拟地址也随机过了，所以需要重定位? */
		handle_relocations(output, output_len, virt_addr);
		debug_putstr("done.\nBooting the kernel.\n");
		return output;
	}

KASLR 的处理入口是 choose_random_location(), 选择随机的物理地址和虚拟地址，定义在 compressed/kaslr.c:

	/*
	 * Since this function examines addresses much more numerically,
	 * it takes the input and output pointers as 'unsigned long'.
	 */
	void choose_random_location(unsigned long input,
				    unsigned long input_size,
				    unsigned long *output,
				    unsigned long output_size,
				    unsigned long *virt_addr)
	{
		unsigned long random_addr, min_addr;

		if (cmdline_find_option_bool("nokaslr")) {
			warn("KASLR disabled: 'nokaslr' on cmdline.");
			return;
		}

	#ifdef CONFIG_X86_5LEVEL
		if (__read_cr4() & X86_CR4_LA57) {
			__pgtable_l5_enabled = 1;
			pgdir_shift = 48;
			ptrs_per_p4d = 512;
		}
	#endif

		boot_params->hdr.loadflags |= KASLR_FLAG;

		/* Prepare to add new identity pagetables on demand. */
		initialize_identity_maps();

		/* Record the various known unsafe memory ranges. */
		mem_avoid_init(input, input_size, *output);

		/*
		 * Low end of the randomization range should be the
		 * smaller of 512M or the initial kernel image
		 * location. 对 kaslr 可用空间的起始地址做个限制
		 */
		min_addr = min(*output, 512UL << 20);

		/* Walk available memory entries to find a random address. */
		random_addr = find_random_phys_addr(min_addr, output_size);
		if (!random_addr) {
			warn("Physical KASLR disabled: no suitable memory region!");
		} else {
			/* Update the new physical address location. */
			/* 经过复杂的计算，终于选中了一个可用的随机物理地址，存起来 */
			if (*output != random_addr) {
				add_identity_map(random_addr, output_size);
				*output = random_addr;
			}

			/*
			 * This loads the identity mapping page table.
			 * This should only be done if a new physical address
			 * is found for the kernel, otherwise we should keep
			 * the old page table to make it be like the "nokaslr"
			 * case.
			 */
			finalize_identity_maps();
		}

		/* Pick random virtual address starting from LOAD_PHYSICAL_ADDR. */
		/* "真正"看懂此函数是需要背景知识的。VO 的链接起始地址(在 linker script 中)是：
		 *
		 *     __START_KERNEL_map + LOAD_PHYSICAL_ADDR
		 *
		 * 根据 Documentation/x86/x86_64/mm.txt 可知，分配给 VO 的虚拟地址范围起始于
		 * __START_KERNEL_map. 综上所述，此函数所描述的虚拟地址随机化其实是 base on
		 * __START_KERNEL_map 的，但代码上省略了这一描述。所以初看感觉虚拟地址的的选择
		 * 范围是 (LOAD_PHYSICAL_ADDR, KERNEL_IMAGE_SIZE)，会觉得很奇怪，实际是基于
		 * 地址 __START_KERNEL_map 的。
		 * Base on 分配给 VO 的地址 __START_KERNEL_map, 随机化后的起始虚拟地址选择范围
		 * 大约是：[LOAD_PHYSICAL_ADDR, KERNEL_IMAGE_SIZE - VO image size], 以
		 * CONFIG_PHYSICAL_ALIGN 为 slot size 单位，同样产生一个随即数，选择一个 slot
		 * 作为起始虚拟地址. 由此可见，随机化前后，虚拟地址的 delta 相对是比较小的，不会
		 * 超过 (KERNEL_IMAGE_SIZE - VO image size).
		 * 理解了这一点，对看懂 handle_relocations 将很有帮助
		 */
		if (IS_ENABLED(CONFIG_X86_64))
			random_addr = find_random_virt_addr(LOAD_PHYSICAL_ADDR, output_size);
		*virt_addr = random_addr;
	}

	/* 此函数注释非常长，省略，在代码中查看。
	 *
	 * 之前发了个错误的 patch： https://lore.kernel.org/patchwork/patch/1037742/，
	 * 因为混淆了 input_size 和 ZO memory image size! input_size 只是压缩文件的
	 * size, 即 .rodata..compressed section 的 size; 而 ZO memory image 包含
	 * .rodata..compressed 和其他所有 sections!
	 *
	 * 另发现：.bss 和 .pgtable section 都不占据文件空间。.bss 无需解释，而 .pgtable
	 * 在 head_64.S 中使用 .section directive 定义时使用了 section type: nobits,
	 * 跟 .bss 一样的 type.
	 *
	 * 为什么对这些不能用来解压的空间做 add_identity_map 呢?
	 */
	static void mem_avoid_init(unsigned long input, unsigned long input_size,
				   unsigned long output)
	{
		unsigned long init_size = boot_params->hdr.init_size;
		u64 initrd_start, initrd_size;
		u64 cmd_line, cmd_line_size;
		char *ptr;

		/*
		 * Avoid the region that is unsafe to overlap during
		 * decompression. 这一段是 ZO relocate 后的运行空间，所以叫 ZO_RANGE
		 */
		mem_avoid[MEM_AVOID_ZO_RANGE].start = input;
		mem_avoid[MEM_AVOID_ZO_RANGE].size = (output + init_size) - input;
		add_identity_map(mem_avoid[MEM_AVOID_ZO_RANGE].start,
				 mem_avoid[MEM_AVOID_ZO_RANGE].size);

		/* Avoid initrd. */
		initrd_start  = (u64)boot_params->ext_ramdisk_image << 32;
		initrd_start |= boot_params->hdr.ramdisk_image;
		initrd_size  = (u64)boot_params->ext_ramdisk_size << 32;
		initrd_size |= boot_params->hdr.ramdisk_size;
		mem_avoid[MEM_AVOID_INITRD].start = initrd_start;
		mem_avoid[MEM_AVOID_INITRD].size = initrd_size;
		/* No need to set mapping for initrd, it will be handled in VO. */

		/* Avoid kernel command line. */
		cmd_line  = (u64)boot_params->ext_cmd_line_ptr << 32;
		cmd_line |= boot_params->hdr.cmd_line_ptr;
		/* Calculate size of cmd_line. */
		ptr = (char *)(unsigned long)cmd_line;
		for (cmd_line_size = 0; ptr[cmd_line_size++];)
			;
		mem_avoid[MEM_AVOID_CMDLINE].start = cmd_line;
		mem_avoid[MEM_AVOID_CMDLINE].size = cmd_line_size;
		add_identity_map(mem_avoid[MEM_AVOID_CMDLINE].start,
				 mem_avoid[MEM_AVOID_CMDLINE].size);

		/* Avoid boot parameters. boot_params 的值是由 RSI 寄存器传递过来*/
		mem_avoid[MEM_AVOID_BOOTPARAMS].start = (unsigned long)boot_params;
		mem_avoid[MEM_AVOID_BOOTPARAMS].size = sizeof(*boot_params);
		add_identity_map(mem_avoid[MEM_AVOID_BOOTPARAMS].start,
				 mem_avoid[MEM_AVOID_BOOTPARAMS].size);

		/* We don't need to set a mapping for setup_data. */

		/* Mark the memmap regions we need to avoid。下方简略分析 */
		handle_mem_options();

	#ifdef CONFIG_X86_VERBOSE_BOOTUP
		/* Make sure video RAM can be used. */
		add_identity_map(0, PMD_SIZE);
	#endif
	}

	/* 专为处理 memmap=, mem=, hugepages 三个参数。*/
	static void handle_mem_options(void)
	{
		...
		/* 着重分析 memmap=。参考 Documentation/admin-guide/kernel-parameters.txt。
		 * nn@ss 指定的区域为可用的 memory range，可用作摆放解压后的 kernel; 其他 3 个
		 * 符号: #$! 指定的空间明显不能使用。同时会记录 mem= 的值, 等同于 memmap= 不指定
		 * ss，表示系统所能使用的 memory size, 表示解压缩后的 kernel 不能超过这个 limit.
		 */
		if (!strcmp(param, "memmap")) {
			mem_avoid_memmap(val);
		} else if (strstr(param, "hugepages")) {
			...
		} else if (!strcmp(param, "mem")) {
			...
		}
	}

	/* 地址随机化用的最小起始地址和解压后的 size 都有了，可以直奔主题了 */
	static unsigned long find_random_phys_addr(unsigned long minimum,
						   unsigned long image_size)
	{
		/* Check if we had too many memmaps. */
		if (memmap_too_large) {
			debug_putstr("Aborted memory entries scan (more than 4 memmap= args)!\n");
			return 0;
		}

		/* Make sure minimum is aligned. */
		/* decompressor 的加载地址在 head_32/64.S 都会对齐到 CONFIG_PHYSICAL_ALIGN，
		 * 但 512M 未必，因为 x86-64 下，CONFIG_PHYSICAL_ALIGN 可以是 2,4,6,..16M,
		 */
		minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);

		/* 暂不考虑 EFI firmware */
		if (process_efi_entries(minimum, image_size))
			return slots_fetch_random();

		process_e820_entries(minimum, image_size);
		return slots_fetch_random();
	}

不考虑 EFI 的情况，所以只看最后两个函数即可。process_e820_entries 内容很简单，遍历 boot param 中的 E820 信息，包装成 mem_vector 形式，交给 process_mem_region 处理：

	/* 入参 entry 是 E820 来的信息，表示 RAM 的地址空间；名字 "entry" 隐含 E820 entry
	 * 的意思。可想而知, 本函数将对各种情况的判断，不是重点且比较繁琐，故省略详细分析.
	 *
	 * 背景知识： 一块等于 image_size 的地址空间，在 kaslr 中被成为一个 slot, 若一块地址
	 * 空间的 size 是 image_size 的数倍，我们则称这块地址空间中有多个 kaslr slot. 可用
	 * 的区域以 slot 计(slot_area_index), 不超过 MAX_SLOT_AREA 个。最终选取解压内核的
	 * 空间时，选的也是 slot.
	 */
	static void process_mem_region(struct mem_vector *entry,
				       unsigned long minimum,
				       unsigned long image_size)
	{
		struct mem_vector region, overlap;
		unsigned long start_orig, end;
		struct mem_vector cur_entry;

		/* 函数逻辑粗略描述:
		 * 作为入参的 "E820" entry 的 range, 必须 overlap (minimum，mem_limit),
		 * 所以先对 entry 表示的地址范围做边界判断优化，放入 cur_entry, 再转放入 region.
		 *
		 * 遍历 region, 看其中有多少个 slot. 因为是选择物理地址，所以 slot 的起始地址须
		 * 向上对齐到 CONFIG_PHYSICAL_ALIGN，但向上对齐后也不能大于这个 entry 的 size,
		 * region.size 太小时可能出现这种情况；向上对齐导致 region.size 也相应减小，所以
		 * 要调整 region.size, 调整后的 size 当然不能小于 image_size.
		 * 调整后的 kaslr region candidate 不能落在 avoided area 中, 但可 overlap,
		 * 若 overlap，则需调整 region.size: 减去 overlap 的部分 size, 剩下的 size
		 * 如果还大于 image_size 则存起来; 没 overlap 的话，则直接存起来...
		 */
	}

	/* 确定了所有可用的 slot，剩下的事情就简单了，选一个即可。返回值是物理地址 */
	static unsigned long slots_fetch_random(void)
	{
		...
		/* 重点是随机选一个不可预测的 slot index */
		slot = kaslr_get_random_long("Physical") % slot_max;
		...
	}

compressed/kaslr_64.c:


	/* Used to track our allocated page tables. */
	static struct alloc_pgt_data pgt_data;

	/* The top level page table entry pointer. */
	static unsigned long top_level_pgt;

	/* __PHYSICAL_MASK_SHIFT 的注释已过时，Documentation/x86/x86_64/mm.txt 的内容
	 * 已 overhaul。它现在被定义为 52，表示 x86_64 平台最大支持 52-bit 的物理地址宽度，
	 * 参考 Intel SDM 3a, chapter 4.1.4 的最后两条：最大物理地址宽度是 52；一般
	 * 情况下, linear-address(或者叫虚拟地址) 宽度是 48(显然是在没有 5-level paging
	 * 的 long mode 下)
	 */
	phys_addr_t physical_mask = (1ULL << __PHYSICAL_MASK_SHIFT) - 1;
	/* 同时注意到，有个相应的 __VIRTUAL_MASK_SHIFT 定义如下：
	 *   #ifdef CONFIG_X86_5LEVEL
	 *     #define __VIRTUAL_MASK_SHIFT	(pgtable_l5_enabled() ? 56 : 47)
	 *   #else
	 *     #define __VIRTUAL_MASK_SHIFT	47
	 *   #endif
	 * 我们已知 long mode 下 linear address(or virtual address) 的位宽是 48(或 57
	 * in 5-level paging)，为什么这里的定义都少了 1？看起来是因为 canonical address
	 * 的原因。i386 下不存在 canonical address 的概念，因为虚拟地址是 32 bits 且全部
	 * 使用；而 x86_64 有 64-bit 虚拟地址且不全部使用，因为 64-bit mode 下要求
	 * 地址必须是 canonical address 的形式，即 address bits 63 through to the
	 * most-significant implemented bit are set to either all ones or all zeros,
	 * 所以其实 the most-significant address bit 其实是不用的。
	 */

	/*
	 * Mapping information structure passed to kernel_ident_mapping_init().
	 * Due to relocation, pointers must be assigned at run time not build time.
	 */
	static struct x86_mapping_info mapping_info;

	/* Locates and clears a region for a new top level page table. */
	void initialize_identity_maps(void)
	{
		/* If running as an SEV guest, the encryption mask is required. */
		/* AMD 的特性，mem_encrypt.S 中的汇编函数。略过 */
		set_sev_encryption_mask();

		/* Exclude the encryption mask from __PHYSICAL_MASK */
		/* AMD 的特性，mem_encrypt.S 中的变量。略过 */
		physical_mask &= ~sme_me_mask;

		/* Init mapping_info with run-time function/buffer pointers. */
		mapping_info.alloc_pgt_page = alloc_pgt_page;
		mapping_info.context = &pgt_data;
		mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sme_me_mask;
		mapping_info.kernpg_flag = _KERNPG_TABLE;

		/*
		 * It should be impossible for this not to already be true,
		 * but since calling this a second time would rewind the other
		 * counters, let's just make sure this is reset too.
		 */
		pgt_data.pgt_buf_offset = 0;

		/*
		 * If we came here via startup_32(), cr3 will be _pgtable already
		 * and we must append to the existing area instead of entirely
		 * overwriting it.
		 *
		 * With 5-level paging, we use '_pgtable' to allocate the p4d page table,
		 * the top-level page table is allocated separately.
		 *
		 * p4d_offset(top_level_pgt, 0) would cover both the 4- and 5-level
		 * cases. On 4-level paging it's equal to 'top_level_pgt'.
		 *
		 * 原以为上面第一段注释 outdated, 仔细分析其 commit 3a94707d7a7b 后发觉并不是。
		 * NOTE: 2016 年还没有 5 级页表的概念. 下面的代码本意只是为了判断前面的代码流程是
		 * 32-bit 还是 64-bit boot protocol, 当加入了 5-level 页表的判断后，原代码就
		 * 不能满足需求了。
		 * 32-bit boot protocol 时，startup_32() 在 _pgtable 处为 64-bit mode 构建
		 * 4-level page table，然后走入 startup_64(), 所以只能使用 _pgtable 处的剩余
		 * 空间来分配所需 identity mapped 页表;
		 * 64-bit boot protocol 时，入口是 startup_64()，此时 bootloader 已经备好
		 * identity mapping 的 page table，但肯定不在 ZO 的 _pgtable 处，所以可以
		 * 使用 _pgtable 处的所有空间来构建额外需要的 page table.
		 * 本函数的注释似乎有点小错误: 函数只是找一块区域存放包含全新映射关系的 page table，
		 * 而不是 new top level page table.
		 * 64-bit boot protocol 时, 如 commit 所述，找到已有的 page table 比较麻烦，
		 * 不如在 _pgtable 处 build 一个全新的页表。
		 */
		top_level_pgt = read_cr3_pa();
		if (p4d_offset((pgd_t *)top_level_pgt, 0) == (p4d_t *)_pgtable) {
			/* 32-bit boot protocol + 4-level paging 才会走到这里 */
			debug_putstr("booted via startup_32()\n");
			pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
			pgt_data.pgt_buf_size = BOOT_PGT_SIZE - BOOT_INIT_PGT_SIZE;
			memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
		} else {
			/* 不仅 64-bit boot protocol，现在 5-level paging 时，也会走到这里，此
			 * debug string 有些 out of date。这时将使用全部区域重新构建所有页表*/
			debug_putstr("booted via startup_64()\n");
			pgt_data.pgt_buf = _pgtable;
			pgt_data.pgt_buf_size = BOOT_PGT_SIZE;
			memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
			top_level_pgt = (unsigned long)alloc_pgt_page(&pgt_data);
		}
	}

linux kernel 页表实现了一套兼容所有 paging mode 的数据结构。5-level paging 以前，kernel 用 PGD -> PUD -> PMD -> PTE : Page 的结构描述所有 paging mode；5-level paging 出现后，在 PGD 和 PUD 中间插入一级 P4D，所以现在是 PGD -> P4D -> PUD -> PMD -> PTE : Page 的结构描述所有 paging mode。此刻尚未熟练掌握，暂且仅从代码角度笨拙的分析 p4d_offset 的实现。kaslr_64.c 中 #include 了 asm/pgtable.h，其中有 p4d_offset 的定义，但是被 #if CONFIG_PGTABLE_LEVELS 包裹，此配置项定义在 arch/x86/Kconfig 中：

	config PGTABLE_LEVELS
	        int
	        default 5 if X86_5LEVEL
	        default 4 if X86_64
	        default 3 if X86_PAE
	        default 2

它在配置阶段是 user invisible 的，也就是说根据环境自动配置。在 64-bit mode 下，只有 2 种情况, CONFIG_PGTABLE_LEVELS 等于 4 或 5。还好头文件的包含关系比较简单，可以肉眼分析。asm/pgtable.h 中 #include 了 asm/pgtable_types.h，这两个头文件中都根据 CONFIG_PGTABLE_LEVELS 的不同分别定义符号和 include 头文件。

先看 CONFIG_PGTABLE_LEVELS = 4 的情况。简单分析可知，此时 p4d_offset 定义在 asm-generic/pgtable-nop4d.h：

	/* 入参 pgd 指向 PGT 中某 entry，即 P4D table 的地址, 返回可以 cover 入参 address
	 * 的 P4D table entry 的地址，所以函数名叫 p4d_offset。
	 * 4-level paging 下没有 p4d，用 kernel 的术语说收 p4d folded into ggd. 此时，
	 * pgd 本身就是第 4 级页表，所以返回入参 pgd
	 */
	static inline p4d_t *p4d_offset(pgd_t *pgd, unsigned long address)
	{
		return (p4d_t *)pgd;
	}

再看 CONFIG_PGTABLE_LEVELS = 5 的情况。此时 p4d_offset 定义在 asm/pgtable.h:

	/* to find an entry in a page-table-directory. */
	static inline p4d_t *p4d_offset(pgd_t *pgd, unsigned long address)
	{
		/* 是否 enable 5-level 的逻辑是：
		 *   - 硬件支持(CR4.LA57)
		 *     &&
		 *   - 软件需要(CONFIG_X86_5LEVEL)
		 */
		if (!pgtable_l5_enabled())
			return (p4d_t *)pgd;
		return (p4d_t *)pgd_page_vaddr(*pgd) + p4d_index(address);
	}

杂乱memo:

 1. 看起来 CONFIG_PGTABLE_LEVELS 不会为 1，虽然理论上有1级页表，在 32-bit paging w/ Page Size Extension 的情况下，但推测 Linux Kernel 并不支持。
 2. 引入 4-level paging 时，引入了 PUD
 3. CONFIG_PGTABLE_LEVELS = 2 时，对应 32-bit paging w/o PSE, 在 linux kernel 中是 PGD -> PTE : Page, 所以会包含头文件 pgtable-nop4d.h, pgtable-nopud.h, pgtable-nopmd.h
 4. CONFIG_PGTABLE_LEVELS = 3 时，对应 PAE paging, 在 linux kernel 中是 PGD -> PMD -> PTE : Page, 所以会包含头文件 pgtable-nop4d.h, pgtable-nopud.h
 5. CONFIG_PGTABLE_LEVELS = 4 时，对应 long mode 64-bit paging, 在 linux kernel 中是 PGD -> PUD -> PMD -> PTE : Page, 所以会包含头文件 pgtable-nop4d.h
 6. https://www.kernel.org/doc/gorman/html/understand/understand006.html
 7. https://lwn.net/Articles/717293/
 8. https://lwn.net/Articles/117749/

终于结束 kaslr 的代码分析，总结一下结果，随机选择了解压物理地址 & VO 的起始虚拟地址, 两个地址都是 CONFIG_PHYSICAL_ALIGN 对齐的，虚拟地址范围是(LOAD_PHYSICAL_ADDR, KERNEL_IMAGE_SIZE), base on __START_KERNEL_map; 物理地址范围是 (min(ZO加载地址, 512M), memory 上限).

一个问题: 配置项 CONFIG_RELOCATABLE 和 CONFIG_RANDOMIZE_BASE 二者有什么不同？
CONFIG_RANDOMIZE_BASE 比较简单，表示 KASLR 特性; 而 CONFIG_RELOCATABLE 在 arch/x86/Kconfig 中的描述是 “Build a relocatable kernel”，其实表示 ZO 和 VO 的加载地址都是 relocatable 的，也就是说，boot loader 可以将 ZO 加载到任意地址，解压缩地址也就从 ZO 加载地址开始。但描述中似乎有错误, CONFIG_RELOCATABLE 并不直接导致 kernel image 中包含 relocation 信息，而是 CONFIG_X86_NEED_RELOCS.  x86_64 下，依赖关系是: CONFIG_X86_NEED_RELOCS <-- CONFIG_RANDOMIZE_BASE <-- CONFIG_RELOCATABLE. 也就是说, !CONFIG_RANDOMIZE_BASE && CONFIG_RELOCATABLE 的情况是存在的。而只有因为 CONFIG_RANDOMIZE_BASE 随机化了 VO 的虚拟地址，才需要重新 relocation.

解压缩完成后，对 decompressed kernel, 即 VO 做 ELF 解析，即函数 parse_elf：

	/* VO 是 elf 格式，所以先 validate ELF header. 然后根据 ELF program header 的
	 * indication 加载 VO 中的 segment. 看后半段函数需要熟悉 ELF 格式. */
	static void parse_elf(void *output)
	{
	#ifdef CONFIG_X86_64
		Elf64_Ehdr ehdr;
		Elf64_Phdr *phdrs, *phdr;
	#else
		Elf32_Ehdr ehdr;
		Elf32_Phdr *phdrs, *phdr;
	#endif
		void *dest;
		int i;

		memcpy(&ehdr, output, sizeof(ehdr));
		if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
		   ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
		   ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
		   ehdr.e_ident[EI_MAG3] != ELFMAG3) {
			error("Kernel is not a valid ELF file");
			return;
		}

		debug_putstr("Parsing ELF... ");

		phdrs = malloc(sizeof(*phdrs) * ehdr.e_phnum);
		if (!phdrs)
			error("Failed to allocate space for phdrs");

		memcpy(phdrs, output + ehdr.e_phoff, sizeof(*phdrs) * ehdr.e_phnum);

		for (i = 0; i < ehdr.e_phnum; i++) {
			phdr = &phdrs[i];

			switch (phdr->p_type) {
			case PT_LOAD:
	#ifdef CONFIG_X86_64
				/* 为什么 p_align 需要 2M 对齐?
				 * man elf 中定义：p_align 的值应是 integral power of two. segment
				 * 在文件中的偏移 p_offset 和虚拟地址 p_vaddr 应对齐到 p_align, 而
				 * p_align(一般情况下等于 page size) 应该对齐到 page size. 所以问题
				 * 变成: 链接时，linker 怎么知道 page size 是多少？
				 * 答案在 arch/x86/Makefile 中的链接选项: "-z max-page-size=0x200000"
				 * 同时参考 MAXPAGESIZE of `info ld`.
				 * 看来 linker 中每个 arch 有 default max page size, 但可以修改它.
				 */
				if ((phdr->p_align % 0x200000) != 0)
					error("Alignment of LOAD segment isn't multiple of 2MB");
	#endif

	#ifdef CONFIG_RELOCATABLE
				/*
				 * 先看 #else 分支。relocatable 时，解压 VO 到地址 output, 按照相同
				 * 逻辑减去相应的 LOAD_PHYSICAL_ADDR. 这个表达有点 tricky，其实是说，
				 * 第一个 PT_LOAD segment 需紧贴 VO 的解压缩地址，其他的 PT_LOAD segment
				 * 的加载情况与 !relocatable 时一致 */
				dest = output;
				dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
	#else
				/*
				 * 非 relocatable 时，必须解压 VO 到地址 LOAD_PHYSICAL_ADDR, 而
				 * segment 的 p_paddr 恰好也被设计为基于此地址，所以，按照 p_paddr
				 * 指示直接 load 即可. */
				dest = (void *)(phdr->p_paddr);
	#endif
				/* load VO 中各 segment 到目的地址 */
				memmove(dest, output + phdr->p_offset, phdr->p_filesz);
				break;
			default: /* Ignore other PT_* */ break;
			}//switch
		}//for

		free(phdrs);
	}

参考: [change alignment of code segment in elf](https://stackoverflow.com/questions/33005638/how-to-change-alignment-of-code-segment-in-elf)

然后是重定位处理(handle_relocations), 因为默认情况下压缩数据中有 vmlinux.relocs 文件, 它的产生定义在 arch/x86/boot/compressed/Makefile： arch/x86/tools/relocs 处理 vmlinux(VO) 得到. handle_relocations 函数专为处理 vmlinux.relocs 中的数据而存在. 但为什么默认会生成 vmlinux.relocs? 是时候对这个故事的来龙去脉做一下梳理:

>KASLR 特性的开启，导致 VO 的虚拟地址也被随机化，所以 VO 编译时使用的虚拟地址已不再有效，对于原编译链接时 relocation 的位置，需要 base on 随机化的新虚拟地址对 VO 再次 relocate，那就需要保留辅助 relocation 的信息，即各 .o 文件中的 relocation section, 用于 KASLR 后的再次 relocation.

背景已经知道，第一件事是保留 relocation 信息，如何保留？查看 vmlinux 的 section header 信息，发现有很多 .rela section, 说明 relocation 信息被保留下来了。本来 vmlinux 是完全链接(relocate)过的可执行文件，不会存在 .rela section, vmlinux 中的 .relaXXX section 是怎么来的？ 查看 arch/x86/Makefile 发现代码：

	ifdef CONFIG_X86_NEED_RELOCS
		LDFLAGS_vmlinux := --emit-relocs
	endif

CONFIG_X86_NEED_RELOCS 仅在 CONFIG_RANDOMIZE_BASE(kaslr) 开启时才打开，且不可配置，即 `make menuconfig` 时找不到 CONFIG_X86_NEED_RELOCS. `man ld` 中对 “--emit-relocs” 的解释：

>Leave relocation sections and contents in fully linked executables.  Post link analysis and optimization tools may need this information in order to perform correct modifications of executables.  This results in larger executables.

这下明白了。relocation 信息保存在 relocaton section 中，在 `man elf` 中有描述，被表示为 relocation entry，其中的 r_offset 字段是我们关心的重点，它表示需要进行 relocation 的具体位置，其解释如下：

>For a relocatable file, the value is the byte offset from the beginning of the section to the storage unit affected by the relocation.  For an executable file or shared object, the value is the virtual address of the storage unit affected by the relocation.

也就是说，vmlinux 中保留的重定位信息是 relocation position 的虚拟地址。有了这些背景知识，再来从头看代码。首先分析工具 relocs，代码不算难，熟悉 ELF 文件格式的话比较容易阅读。总的来说：relocs 工具读取 vmlinux 文件中的所有 section，过滤出 relocation section，将所有 relocation entry 中的 r_offset 字段数据保留下来，输出到 vmlinux.relocs 文件，用于后续 KASLR 的 relocation processing 使用。

>Side note: 通过 `readelf -S` 查看 vmlinux 或者 .o 文件 section header 信息会发现，每个 PROGBITS 类型的 section 都会有其对应的 .rela section, 而且在 section header table 中紧相邻。也会发现 .data section 也有相应的 .rela section, 乍一看比较奇怪，因为数据区好像不存在符号引用的问题，实际上是因为数据区可能有全局指针变量，所以才会需要 relocation, 可以通过简单的小程序来测试。

relocs 的大部分代码比较普通，没有难度，分析一下部分比较 tricky 的代码：

	static int do_reloc64(struct section *sec, Elf_Rel *rel, ElfW(Sym) *sym,
		      const char *symname)
	{
		unsigned r_type = ELF64_R_TYPE(rel->r_info);
		/* 这里 offset 指 VO 中发生重定位的虚拟地址处 */
		ElfW(Addr) offset = rel->r_offset;
		...

		/*
		 * Adjust the offset if this reloc applies to the percpu section.
		 * percpu section 还不理解，待分析。
		 */
		if (sec->shdr.sh_info == per_cpu_shndx)
			offset += per_cpu_load_addr;

		switch (r_type) {
		case ...

		case R_X86_64_32:
		case R_X86_64_32S:
		case R_X86_64_64:
			...
			/*
			 * Relocation offsets for 64 bit kernels are output
			 * as 32 bits and sign extended back to 64 bits when
			 * the relocations are processed.
			 * Make sure that the offset will fit.
			 */
			/* 没有背景知识的话，很难理解上面的 comments. 查看 VO 的 linker script
			 * arch/x86/kernel/vmlinux.lds.S 和 Documentation/x86/x86_64/mm.txt
			 * 可知, x86_64 下，VO(kernel) 的虚拟地址被安排在 ffffffff80000000 起始
			 * 的范围(__START_KERNEL_map), 除了 .data..percpu section 的虚拟地址从 0
			 * 开始, 而 .data..percpu section 的 size 很小(0x23000 bytes 在我的环境)。
			 * 所以 r_offset 的值只可能有 2 种样子: 0xffffffff 8xxxxxxx 或
			 * 0x00000000 000xxxxx, 这样的话，我们可以只保存它的低 32 bits，使用的时候
			 * 在 sign extended back to 64 bits. 之前理解困难，是因为最基础的知识欠缺：
			 * 有符号整数的扩展. 比如: 将 32 bits 负数 -249346713(0xF1234567) sign
			 * extend 到 64 bits 是 0xFFFFFFFF F1234567.
			 * 对 offset 的 if 判断，就是保证 address form 是上述的样子，只有这样的
			 * address 才能安全 sign extend back 且数值不会改变.
			 * 这 3 种 relocation type 的地址计算方式都是 S(symbol value) + A(addend),
			 * 即直接在需要 relocation 的位置填入 reference 的目标符号的地址即可
			 * */
			if ((int32_t)offset != (int64_t)offset)
				die("Relocation offset doesn't fit in 32 bits\n");

			if (r_type == R_X86_64_64)
				add_reloc(&relocs64, offset);
			else
				add_reloc(&relocs32, offset);
			break;
		}

		return 0;
	}

Simply speaking: relocs 工具把 vmlinux 中出现的几种 relocation type 是 (S + A) 的 relocation entry 的 r_offset(relocation 发生的地址) 字段保存到 vmlinux.relocs 文件. 可想而知，若虚拟地址被 KASLR 随机化，原来 relocation 位置中的符号地址，要根据随机化后的新地址相应的更新，这就是 handle_relocations 函数要做的事情。理解这个函数的细节，需要了解一个**很重要的逻辑**: 不管是从 vmlinux 的加载(物理)地址，还是链接的虚拟地址，还是 vmlinux 文件内的偏移看，relocation 的位置相对起始点的 offset 是不变的，即: relocation 的 file offset = relocation 的物理地址 - vmlinux 物理起始地址 = relocation 的虚拟地址 - vmlinux 的虚拟起始地址。

理解了上述逻辑，就容易理解 *handle_relocations* 了：

	#if CONFIG_X86_NEED_RELOCS
	static void handle_relocations(void *output, unsigned long output_len,
				       unsigned long virt_addr)
	{
		int *reloc;
		unsigned long delta, map, ptr;
		unsigned long min_addr = (unsigned long)output;
		unsigned long max_addr = min_addr + (VO___bss_start - VO__text);

		/* 局部变量 min_addr/max_addr 的含义: relocation 的位置当然在 VO 的 memory
		 * image 内，也即 VO 被解压的物理地址范围(.bss section 不需 relocation). 这
		 * 是本函数的核心内容：根据已知信息，找到 VO memory image 中需要再次 relocation
		 * 的物理地址, 将原虚拟地址和新虚拟地址的 delta, apply 到需 relocation 的位置.
		 */

		/*
		 * Calculate the delta between where vmlinux was linked to load
		 * and where it was actually loaded.
		 */
		delta = min_addr - LOAD_PHYSICAL_ADDR;

		/*
		 * The kernel contains a table of relocation addresses. Those
		 * addresses have the final load address of the kernel in virtual
		 * memory. We are currently working in the self map. So we need to
		 * create an adjustment for kernel memory addresses to the self map.
		 * This will involve subtracting out the base address of the kernel.
		 */
		map = delta - __START_KERNEL_map;

		/*
		 * 32-bit always performs relocations. 64-bit relocations are only
		 * needed if KASLR has chosen a different starting address offset
		 * from __START_KERNEL_map.
		 */
		if (IS_ENABLED(CONFIG_X86_64))
			delta = virt_addr - LOAD_PHYSICAL_ADDR;

		if (!delta) {
			debug_putstr("No relocation needed... ");
			return;
		}
		debug_putstr("Performing relocations... ");

		/* 上面几个变量的算术运算，单独看的话理解有困难，不妨结合下面的代码展开：
		 *
		 * extended = extended(虚拟地址) + map
		 * 			= extended(虚拟地址) + delta - __START_KERNEL_map
		 * 			= extended(虚拟地址) + min_addr - LOAD_PHYSICAL_ADDR - __START_KERNEL_map
		 * 			= extended(虚拟地址) - (LOAD_PHYSICAL_ADDR + __START_KERNEL_map) + min_addr
		 *
		 * 等式左边的 extended 是待求值的需再次重定位的物理地址； 右边的 extended 是
		 * vmlinux.relocs 中 relocation entry 中的值，表示编译时发生重定位的虚拟地址；
		 * (LOAD_PHYSICAL_ADDR + __START_KERNEL_map) 是 VO 的链接起始虚拟地址，extended
		 * 减去它得到上面说的待再次 relocation 位置在 file image 中的 offset; 将不变的
		 * offset 加到 VO 的物理地址 min_addr 上，便得到 VO 的 memory image 中需再次
		 * relocation 的地址。
		 */

		/*
		 * Process relocations: 32 bit relocations first then 64 bit after.
		 * Three sets of binary relocations are added to the end of the kernel
		 * before compression. Each relocation table entry is the kernel
		 * address of the location which needs to be updated stored as a
		 * 32-bit value which is sign extended to 64 bits.
		 *
		 * Format is:
		 *
		 * kernel bits...
		 * 0 - zero terminator for 64 bit relocations
		 * 64 bit relocation repeated
		 * 0 - zero terminator for inverse 32 bit relocations
		 * 32 bit inverse relocation repeated
		 * 0 - zero terminator for 32 bit relocations
		 * 32 bit relocation repeated
		 *
		 * So we work backwards from the end of the decompressed image.
		 */
		for (reloc = output + output_len - sizeof(*reloc); *reloc; reloc--) {
			long extended = *reloc; /* sign extend back 得到完整的重定位地址 */
			extended += map; /* 获得要做 relocation 的物理地址 */

			/* relocation 的物理地址必须在 VO 的加载物理地址范围内 */
			ptr = (unsigned long)extended;
			if (ptr < min_addr || ptr > max_addr)
				error("32-bit relocation outside of kernel!\n");

			/* 物理地址处的内容是被引用符号的链接时虚拟地址，加上随机化后虚拟地址的 delta
			 * 即得到符号的新虚拟地址。处理 32 bit relocation，所以是 uint32_t. */
			*(uint32_t *)ptr += delta;
		}
	#ifdef CONFIG_X86_64
		while (*--reloc) {
			long extended = *reloc;
			extended += map;

			ptr = (unsigned long)extended;
			if (ptr < min_addr || ptr > max_addr)
				error("inverse 32-bit relocation outside of kernel!\n");

			/* 不明白，TBD. */
			*(int32_t *)ptr -= delta;
		}
		for (reloc--; *reloc; reloc--) {
			long extended = *reloc;
			extended += map;

			ptr = (unsigned long)extended;
			if (ptr < min_addr || ptr > max_addr)
				error("64-bit relocation outside of kernel!\n");

			*(uint64_t *)ptr += delta;
		}
	#endif
	}


#### string operation under x86/boot

此 topic 的引入，源自研究 x86/boot/string.{c,h} 时的发现。x86/boot/ 下的 string.{c.h} 不仅在所在目录中使用，而且在 x86/boot/compressed，x86/purgatory 目录下使用。本来意图很简单，一些通用的字符操作，无需多次定义。但在实际使用中，发现了一些有趣的小现象。

对于 memcpy/memset/memcmp，x86/boot/string.h 做了声明，同时定义了 macro：

	void *memcpy(void *dst, const void *src, size_t len);
	void *memset(void *dst, int c, size_t len);
	int memcmp(const void *s1, const void *s2, size_t len);

	#define memcpy(d,s,l) __builtin_memcpy(d,s,l)
	#define memset(d,c,l) __builtin_memset(d,c,l)
	#define memcmp	__builtin_memcmp

x86/boot/string.c 中 #undef 了上述宏，同时定义了 memcmp(没定义 memset, memcpy)；而 x86/boot/copy.S 以汇编代码定义了 memset 和 memcpy。同时要注意，编译 x86/boot/ 下 setup 文件时，使用了选项 `-ffreestanding`，它在 [GCC 文档](https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html#C-Dialect-Options)中的定义如下：

>Assert that compilation targets a freestanding environment. This implies
‘-fno-builtin’. bluhbluh...

`-fno-builtin` 的定义也在上述 GCC 文档：

>Don’t recognize built-in functions that do not begin with ‘__builtin_’ as prefix. bluhbluh...

Simply speaking，标准 C 库中的很多函数都有其 GCC builtin 版本，这些 GCC builtin 函数有两种形式，带前缀 "__builtin_" 和不带的，使用选项 `-fno-builtin` 意味着不带前缀则不会被识别为 builtin 函数。继续阅读之前需要补充一些背景知识：

 1.[C Language Standards supported by GCC](https://gcc.gnu.org/onlinedocs/gcc/Standards.html#C-Language)

 2.[Other Built-in Functions Provided by GCC](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#Other-Builtins)

实际使用中，x86/boot/ 下的 .c 文件会 #include "string.h"，也就是说会使用 memxxx() 的 GCC builtin 版本。这么说来，x86/boot 下定义的 memxxx() 不会被用到？这要 case by case 的看：

  * memcmp() 定义在 string.c，被同文件中的 strstr()使用；
  * memcmp() 定义在 copy.S，被同文件中的 copy_from_fs/copy_to_fs 使用；
  * memset() 定义在 copy.S，但看起来没有被特别的使用，因为凡使用 memset 的文件头部都有 #include "string.h"，所以看起来 copy.S 中定义的 memset() 没有被使用，从 setup 的 `objdump -d` 输出中也可以确认这一点。

memset() 的情况引出了 [patch](https://lkml.org/lkml/2019/1/7/60)。x86 maintainer 的回复太简洁，无法明白背后的原理。发了 patch 后，又发现 x86/boot/compressed 目录下的[有趣现象](https://lkml.org/lkml/2019/1/8/128)，实际上和 patch 是一个问题。最后，从 GCC 社区得到了[帮助](https://gcc.gnu.org/ml/gcc-help/2019-01/msg00039.html)。

简而言之，当 GCC 看到 builtin 函数后，会使用启发式的策略决定如何 expand 它：emit a call to library equivalent(我们的情况下就是 call 自己定义的函数), 还是 optimize to inline code。有编译选项来精确的控制 expand 行为：`-mstringop-strategy=`, `-mmemcpy-strategy=`, `-mmemset-strategy=strategy`. 从这些选项的释义可以看出，可以配置是否 inline builtin 函数，inline 时根据操作数 size 选择不同的 inline 算法。

经过测试，`-mstringop-strategy=` 的确可以达成期望。在 arch/x86/boot/compressed/Makefile 添加：

	$(obj)/kaslr.o: KBUILD_CFLAGS += -mstringop-strategy=byte_loop
	$(obj)/pgtable_64.o: KBUILD_CFLAGS += -mstringop-strategy=libcall

后编译，观察 kaslr.o 和 pgtable_64.o 的 nm 输出，并 `grep mem*`，就可确认编译选项生效，与 hack 之前的结果相反，符合预期。

#### VO/ZO

head_64.S 和 head_32.S 中有用到 macro: BP_init_size，它表示 header.S 中的 boot protocol field: init_size。为了理解用到它的代码，必须搞清楚它的真实含义。init_size 在 header.S 中被定义为：

	init_size:		.long INIT_SIZE		# kernel initialization size

你会发现 INIT_SIZE 的定义非常复杂，牵扯了一堆 ZO_、VO_ 开头的变量，这些变量分别定义在 arch/x86/boot/ 下的 zoffset.h 和 voffset.h 中，若不理解 arch/x86/boot/vmlinux.bin 的处理流程，很难理解这些变量的含义。所以，是时候兑现上文的诺言了。kernel 文档中没有发现对 VO/ZO 的官方权威解释，唯一的解释在 [patch](https://lore.kernel.org/patchwork/patch/674100/) 中。

相关文件的处理过程定义在 Makefile 中，技术细节属于 kbuild 领域，本文不展开解释，仅直接告诉结果：源码根目录下的 vmlinux 被 `objcopy -R .comment -S` 为 arch/x86/boot/compressed/vmlinux.bin；vmlinux.bin(和可选的vmlinux.relocs 一起) 被压缩为 vmlinux.bin.gz(默认压缩算法)，作为同目录下 host program `mkpiggy` 的输入，生成 piggy.S；piggy.S 和同目录的其他源代码文件一起编译生成该目录下的 vmlinux；此 vmlinux 被 objcopy 剥离为上一层目录的 vmlinux.bin，即 arch/x86/boot/vmlinux.bin，此 vmlinux.bin 与同目录的 setup.bin 一起被 host program `build` 打包在一起成 bzImage。

piggy.S 由 `mkpiggy` 生成，有必要看一下 `mkpiggy` 做了什么。[mkpiggy 的源代码](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/mkpiggy.c)很简单，但需要结合 [gzip spec](https://tools.ietf.org/html/rfc1952) 才能理解。

>gzip spec tips: 一个 gzip 文件由一系列 members (compressed data sets) 组成，因为可以多个文件被压缩在一起，所以一个 member 表示一个被压缩的文件；而 kernel 压缩是以 `cat file | gzip` 的方式，所以只有 member。多字节整数在 gzip 压缩文件中以 Little-endian 存放。

目前，linux kernel 支持 6 种压缩算法：`.gz`, `.bz2`, `.lzma`, `.xz`, `.lzo`, `.lz4`，为了直观感受 objcopy 的剥离效果，以及各种压缩算法的效果，在我的测试环境下，各相关文件 size 如下：

	# before stripped
	[pino@IAAS0 linux]$ ll -h vmlinux
	-rwxrwxr-x 1 pino pino 576M Nov 10 12:01 vmlinux

	[pino@IAAS0 linux]$ ll -h arch/x86/boot/compressed/vmlinux.bin*
	-rwxrwxr-x 1 pino pino  30M Nov 10 12:02 arch/x86/boot/compressed/vmlinux.bin
	-rw-rw-r-- 1 pino pino 8.3M Nov 10 11:51 arch/x86/boot/compressed/vmlinux.bin.bz2
	-rw-rw-r-- 1 pino pino 7.9M Nov  9 19:29 arch/x86/boot/compressed/vmlinux.bin.gz
	-rw-rw-r-- 1 pino pino  11M Nov 10 13:37 arch/x86/boot/compressed/vmlinux.bin.lz4
	-rw-rw-r-- 1 pino pino 6.2M Nov 10 12:28 arch/x86/boot/compressed/vmlinux.bin.lzma
	-rw-rw-r-- 1 pino pino 9.4M Nov 10 12:40 arch/x86/boot/compressed/vmlinux.bin.lzo
	-rw-rw-r-- 1 pino pino 5.8M Nov 10 12:35 arch/x86/boot/compressed/vmlinux.bin.xz

岔开了一点话题，继续看 mkpiggy 如何处理 arch/x86/boot/compressed/vmlinux.bin。由 spec 上 member format 可知，每个 member 的最后 4 bytes 是该文件压缩前的 size：

	ISIZE (Input SIZE)
		This contains the size of the original (uncompressed) input
		data modulo 2^32.

如代码所示，这也是 mkpiggy 想要读出的信息：

	# mkpiggy.c
	...
	if (fseek(f, -4L, SEEK_END)) {
		perror(argv[1]);
	}

	/* fread 读取数据后会相应移动 file stream 的 position indicator，所以读完后，
	 * position indicator 在文件尾，下面的的 ftell 读出的就是整个文件的 size */
	if (fread(&olen, sizeof(olen), 1, f) != 1) {
		perror(argv[1]);
		goto bail;
	}

	ilen = ftell(f);
	olen = get_unaligned_le32(&olen); /* 为什么? */
	...

6 种压缩算法中，gzip 格式天然支持在压缩文件末尾附上文件压缩前的 size，其他压缩算法都是通过 Makefile 中的 `size_append` 操作在压缩文件末尾以 little-endian 附上压缩前的 size。mkpiggy 仅在 x86 上使用，x86 是 little-endian CPU，所以，为什么需要 endian 转换？ x86 maintainer 给了[解释](https://lkml.org/lkml/2018/11/9/1166)，因为考虑到了在 big-endian 机器上交叉编译 x86 的 kernel = =|，不知道谁有这种使用场景需求。

生成的 piggy.S，在我的测试环境中长这样：

	.section ".rodata..compressed","a",@progbits
	.globl z_input_len
	z_input_len = 8227726
	.globl z_output_len
	z_output_len = 31329060
	.globl input_data, input_data_end
	input_data:
	.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
	input_data_end:

它记录了内核压缩前后的 size，把压缩后的 kernel 放在一个独立的 section 中。

Tip: 汇编语言中定义的符号(label)表示 location counter 的当前值，也就是地址，给符号赋值也是表示改变其地址，这里把本表示 size 的值赋值给符号。汇编代码中使用符号时是使用其地址值。参考 "5.5.1 Value" of `info as`.

zoffset.h 和 voffset.h 两个文件在编译过程中生成，只有了解他们的生成细节，才能理解文件中那些变量的含义。arch/x86/boot/voffset.h 由 arch/x86/boot/compressed/Makefile 定义:

	sed-voffset := -e 's/^\([0-9a-fA-F]*\) [ABCDGRSTVW] \(_text\|__bss_start\|_end\)$$/\#define VO_\2 _AC(0x\1,UL)/p'

	quiet_cmd_voffset = VOFFSET $@
	      cmd_voffset = $(NM) $< | sed -n $(sed-voffset) > $@

	targets += ../voffset.h

	$(obj)/../voffset.h: vmlinux FORCE
	        $(call if_changed,voffset)

voffset.h 由 nm 处理源码根目录下的 vmlinux 而来(代码细节属于 kbuild 领域，本文不展开)。nm 用于列出目标文件中的符号，对于每一个符号，列出其 symbol value(address), symbol type, symbol name，它的输出长这样：

	000000000000005d t try_to_run_init_process

sed 的用法参考 `info sed`。这条 sed script 过滤出 vmlinux 中的三个符号(_text,__bss_start,_end)的地址，输出到文件 voffset.h，这三个符号都定义在 vmlinux 的 linker script(arch/x86/kernel/vmlinux.lds)中。在我的例子中，voffset.h 长这样：

	#define VO___bss_start _AC(0xffffffff82919000,UL)
	#define VO__end _AC(0xffffffff82a7e000,UL)
	#define VO__text _AC(0xffffffff81000000,UL)

可以推断：前缀 VO 的 V 表示 Vmlinux，O 猜测是 Object 或 Offset。_text 表示 vmlinux 的起始地址，_end 表示 vmlinux 的结束地址，__bss_start 的地址在二者之间。

arch/x86/boot/zoffset.h 由 arch/x86/boot/Makefile 定义：

	sed-zoffset := -e 's/^\([0-9a-fA-F]*\) [ABCDGRSTVW] \(startup_32\|startup_64\|efi32_stub_entry\|efi64_stub_entry\|efi_pe_entry\|input_data\|_end\|_ehead\|_text\|z_.*\)$$/\#define ZO_\2 0x\1/p'

	quiet_cmd_zoffset = ZOFFSET $@
	      cmd_zoffset = $(NM) $< | sed -n $(sed-zoffset) > $@

	targets += zoffset.h
	$(obj)/zoffset.h: $(obj)/compressed/vmlinux FORCE
	        $(call if_changed,zoffset)

可以看出，处理流程与 voffset.h 完全一致，只不过过滤出更多的符号地址在 zoffset.h 中。我的环境中，zoffset.h 长这样：

	#define ZO__ehead 0x00000000000003b4
	#define ZO__end 0x00000000007ee000
	#define ZO__text 0x00000000007b6ce0
	#define ZO_efi32_stub_entry 0x0000000000000190
	#define ZO_efi64_stub_entry 0x0000000000000390
	#define ZO_efi_pe_entry 0x00000000000002c0
	#define ZO_input_data 0x00000000000003b4
	#define ZO_startup_32 0x0000000000000000
	#define ZO_startup_64 0x0000000000000200
	#define ZO_z_input_len 0x00000000007b6927
	#define ZO_z_output_len 0x0000000001e04eec

可以推断：前缀 ZO 中的 Z 表示压缩后。理解了这些，再来分析 header.S 中的 INIT_SIZE 的含义：

	/* 为方便阅读，格式有做优化。源文件中本段代码上面有大段注释解释为什么需要 extra bytes，
	 * 不在作者的知识领域，我们仅需知道：为了 safety decompression in place, 解压缩
	 * buffer 的大小是原文件 size + extra bytes。以我的环境为例: 1e04eec >> 8 + 65536
	 * = 188494 bytes, 约等于 184k。
	 */
	#define ZO_z_extra_bytes	((ZO_z_output_len >> 8) + 65536)

	#if ZO_z_output_len > ZO_z_input_len
		/* 这是正常情况，没听过压缩后比压缩前文件 size 还大。压缩文件放在解压缩 buffer 的
		 * 尾端，此宏表示它在 buffer 中的 offset. 本例中其值约为：30M + 184k - 7.7M,
		 * 约等于 22.5M */
	    # define ZO_z_extract_offset	(ZO_z_output_len + ZO_z_extra_bytes - \
						 				ZO_z_input_len)
	#else
		/* 由 https://lore.kernel.org/patchwork/patch/674100/ 可知，压缩后比压缩前
		 * 大的情况 uncommon but possible */
	    # define ZO_z_extract_offset	ZO_z_extra_bytes
	#endif

	/*
	 * The extract_offset has to be bigger than ZO head section. Otherwise when
	 * the head code is running to move ZO to the end of the buffer, it will
	 * overwrite the head code itself.
	 */
	#if (ZO__ehead - ZO_startup_32) > ZO_z_extract_offset
	    /* 二者相减是 ZO 中的 .head.text section 的 size。本段代码其他信息参考:
	     * https://lore.kernel.org/patchwork/patch/674095 */
	    # define ZO_z_min_extract_offset ((ZO__ehead - ZO_startup_32 + 4095) & ~4095)
	#else
	    /* 正常情况下，只将 extract offset 向上对齐到 4k 边界，基本还是 22.5M */
	    # define ZO_z_min_extract_offset ((ZO_z_extract_offset + 4095) & ~4095)
	#endif

	/* 前两个变量相减表示 arch/x86/boot/compressed/vmlinux 的 memory image size,
	 * 表达式的值约等于 30.4M */
	#define ZO_INIT_SIZE	(ZO__end - ZO_startup_32 + ZO_z_min_extract_offset)

	/* VO 在内存中的 size, 本例中约等于 26.5M! 原来的认知被颠覆了 = =! */
	#define VO_INIT_SIZE	(VO__end - VO__text)

	/* 谁大选谁。认知颠覆后，本例中就是 ZO_INIT_SIZE, 这样看来，extract_kernel 注释中的
	 * 图示开始 make sense */
	#if ZO_INIT_SIZE > VO_INIT_SIZE
		# define INIT_SIZE ZO_INIT_SIZE
	#else
		# define INIT_SIZE VO_INIT_SIZE
	#endif

现在回头看 Documentation/x86/boot.txt 中 init_size 的定义，也发现开始 make sense 了。可以不用纠结 INIT_SIZE 的计算过程，只需要知道需要这么一块 memory 作 buffer，来 in-place decompression, 这块 memory 的起始地址是 kernel 的 runtime start address。

github 上电子书 《Linux Inside》 有一个 [issue](https://github.com/0xAX/linux-insides/issues/597), 虽然提问内容不太对, 但还是触发了一个问题(我猜很少人会想到)：解压缩完毕后，会使用 `jmp	*%rax` 跳到 VO 中执行，这条指令 jmp 不会覆盖吗? 这个问题需要结合多个 facts：上面的实例数据，更上面的 bzImage file layout 图示，代码细节。buffer size 是 ~30.4M; 解压数据中 VO memory size 是 ～26.5M, vmlinux.relocs 是 ～0.8M, 总计 ~27.3M; ZO memory size 是 ～7.9M, 几乎全被 .rodata..compressed(压缩数据占据)，紧贴 buffer 底部摆放，重点是！解压代码(extract_kernel) 和 label: relocated 都位于 .text(这点不容易注意到)，在 .rodata..compressed 之后,  .head.text 和 .rodata..compressed 部分空间会被覆盖, .text 不会被解压数据覆盖，所以 jmp 指令不会被覆盖.


#### linker script

分析链接 kernel 用的 linker script 对于理解 linux kernel 的代码是很有帮助的。setup, decompressor 和 VO 各有其 linker script，前两者比较简单，VO 就复杂很多。

学习 GUN linker script 的最佳方式应该还是阅读它的官方文档 via `info ld`，这应该是最权威的文档。但个人阅读下来感受到一个小问题： linker script 语言元素的 hierarchy 略多，而且许多细节描述夹杂其中，阅读中经常会忘记层次并迷失在其中。故这里用简单的语言总结下 linker script 的格式： 链接脚本的内容是一系列命令，这些命令可能是 linker script 的关键字(可能带参数)，也可能是赋值语句；命令之间可以用分号隔开(也可以不用，换行即可)。常见的命令关键字如：

>ENTRY(SYMBOL)

>OUTPUT_FORMAT(DEFAULT, BIG, LITTLE)

>OUTPUT_ARCH(BFDARCH)

>**SECTIONS**

>PHDRS

>ASSERT(EXP, MESSAGE)

等等。其中 **SECTIONS** 命令是最重要的，它描述了如何把 input section 输出(layout)到 output section，以及这些 output section 在 memory 中的地址。**SECTIONS**命令的内容包含了一些列 SECTIONS-COMMAND，SECTIONS-COMMAND 又可以分为几类内容：

>ENTRY command (很少见)

>symbol assignment (常见)

>output section description (几乎必见，继续介绍它)

>overlay description (很少见)

output section description 的完整描述长这样：

     SECTION [ADDRESS] [(TYPE)] :
       [AT(LMA)]
       [ALIGN(SECTION_ALIGN) | ALIGN_WITH_INPUT]
       [SUBALIGN(SUBSECTION_ALIGN)]
       [CONSTRAINT]
       {
         OUTPUT-SECTION-COMMAND
         OUTPUT-SECTION-COMMAND
         ...
       } [>REGION] [AT>LMA_REGION] [:PHDR :PHDR ...] [=FILLEXP] [,]

其中 OUTPUT-SECTION-COMMAND 又可以分为几类：

>symbol assignment (常见)

>input section description (必见)

>data values to include directly (偶尔见)

>special output section keyword (很少见)

command(如 ASSERT) 有时也会出现在 output/input section description 中。

三层语言要素中，名字重复多，不同的层次的语言元素也有相似之处，所以初看时易迷失。

linker script 中可以定义符号，详见 “3.5 Assigning Values to Symbols” of `info ld`. 定义符号的形式很简单：

	SYMBOL = EXPRESSION;

符号的值表示的是地址。 script 中定义的符号可以在高级语言的源代码中引用，详细描述在 "3.5.5 Source Code Reference" of `info ld`(强烈推荐阅读).

Tip: linker script 中提到的地址，对于 x86 来说，不论是 Virtual Memory Address(VMA) 还是 Load Memory Address(LMA)，都在 x86 的内存分段模型中，也就是说，这些地址是 code segment 中的地址。

X86 下链接 vmlinux 用的 script 是 arch/x86/kernel/vmlinux.lds.S，但这只是原始模板，它会被 C Preprocessor(cpp) 处理为 arch/x86/kernel/vmlinux.lds，后者才是真正使用的 linker script，但是因为被 cpp 处理过，所以格式上不 Human-friendly。

#### To Be Done

 1. UEFI 启动分析，EFI handover protocol, 即 X86-64 CPU 上，32-bit EFI firmware 启动 64-bit kernel
 2. 使用 ld version < 2.24 编译，查看 decompressor 中得 GOT 前三个 entry

## VO/The "real" Linux kernel

为什么强调 "real" 呢？虽然前面分析的代码也属于 Linux kernel, 但并不是绝大多数 developer 的 coding area. 通常我们在说 kernel development 时，主要指的 VO 部分，因为源码根目录下的绝大多数文件属于 VO. 为了保证尽可能简单的分析流程，理解最重要的 steps, 我们将忽略一些不重要的 branch feature, 无特殊说明时，我们只分析 x86_64 代码, 将忽略： SME(AMD's Secure Memory encryption), 5-level paging, PTI(Page Table Isolation), XEN.

### vmlinux linker script

linker script 很少是 developer 的焦点，但有时会在这里找到一些问题的答案。学习 linker script 的权威材料在 `info ld`. 对 x86 来说，它的 memory layout 由 arch/x86/kernel/vmlinux.lds.S 指定。

#### jiffies & jiffies_64
vmlinux.lds.S 的开头就有个有趣的小问题，代码如下(微调了格式，方便阅读)：

	#ifdef CONFIG_X86_32
		OUTPUT_ARCH(i386)
		ENTRY(phys_startup_32)
		jiffies = jiffies_64;
	#else
		OUTPUT_ARCH(i386:x86-64)
		ENTRY(phys_startup_64)
		jiffies_64 = jiffies;
	#endif

问题来了: jiffies 和 jiffies_64 在 X86_32 和 X86_64 分别是如何定义的？

X86_32 下，script 中有:

	jiffies = jiffies_64;

定义了符号 jiffies，它的值(地址)等于 jiffies_64，而 jiffies_64 定义在 kernel/time/timer.c 中：

	__visible u64 jiffies_64 __cacheline_aligned_in_smp = INITIAL_JIFFIES;

所以，script 中定义的符号 reference 了代码中的符号。

X86_64 下，script 中有：

	jiffies_64 = jiffies;

而源代码中(arch/x86/kernel/time.c)有：

	#ifdef CONFIG_X86_64
	__visible volatile unsigned long jiffies __cacheline_aligned_in_smp = INITIAL_JIFFIES;
	#endif

说明 jiffies 和 jiffies_64 两个符号在代码中都有定义，又因为 script 中符号的值表示其地址，所以 script 中的赋值操作表示这两个符号的地址相等，通过 readelf 读取两个符号的值可以确认：

>$ readelf -s vmlinux | grep -w jiffies
 12797: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS jiffies.c
 84387: ffffffff82205000     8 OBJECT  GLOBAL DEFAULT   24 jiffies
$ readelf -s vmlinux | grep -w jiffies_64
 82843: ffffffff82205000     8 OBJECT  GLOBAL DEFAULT   24 jiffies_64

#### linking address of VO

VO 的 linker script 中涉及了 VMA(virtual memory address) 和 LMA(load memory address) 两种地址.

将 linker script command `SECTIONS` 的开头两行展开(x86_64 下)：
```
	. = __START_KERNEL;
	phys_startup_64 = ABSOLUTE(startup_64 - LOAD_OFFSET);
```
==>
```
. = __START_KERNEL_map + ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN);
phys_startup_64 = startup_64 - __START_KERNEL_map;
```
==>
```
. = 0xffffffff80000000 + ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN);
phys_startup_64 = ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN);
```
(这里快进了一步)startup_64 定义在 arch/x86/kernel/head_64.S 的开头，表示 VO 的入口地址。由上述展开结果可知，VO 的 VMA = __START_KERNEL_map + offset, 参考 Documentation/x86/x86_64/mm.txt 可知，分配给 kernel 的地址空间从 __START_KERNEL_map 开始。

第一个 output section definition 是：
```
.text :  AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	_stext = .;
	...
}
```
AT() 表示 .text section 的 LMA, 即物理地址。AT() 中的表达式转换到最后也等于
```
ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```
这里出现的 VMA(virtual memory address) 和 LMA(load memory address) 定义，在下文分析会经常 reference.

head_64.S 和 head64.C 中经常看到一种求某符号编译时物理地址(LMA in lds)的表达式：

    physical address of symbol A = virtual address of symbol A - VO linkage start address

为什么会有这种关系呢? vmlinux.lds 中的特殊地址设计，使得 VO 的 LMA 和 VMA 分别是：

>物理地址 [0 + **offset** - KERNEL_IMAGE_SIZE]

>虚拟地址 [0xffffffff80000000 + **offset** - KERNEL_IMAGE_SIZE]

且 **offset** = ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)

所以, 符号的物理地址 = 符号虚拟地址 - 0xffffffff80000000. 表面看, 减法结果表示偏移，但这个偏移值在虚拟地址和物理地址中表示的意义一样，即符号在 VO image 中的 offset, 又因为 VO 物理地址从地址 0 开始，所以此时 offset 也是物理地址。若 KASLR 开启或者 kernel 配置为 relocatable 时导致物理地址不同于 ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)，在 __startup_64 函数中会根据物理地址的 load delta 调整 head_64.S 中构建的页表。

#### PHDRS
script 中通过 `PHDRS` 定义了自己的 Program Header:
```
PHDRS {
	text PT_LOAD FLAGS(5);          /* R_E */
	data PT_LOAD FLAGS(6);          /* RW_ */
#ifdef CONFIG_X86_64
#ifdef CONFIG_SMP
	percpu PT_LOAD FLAGS(6);        /* RW_ */
#endif
	init PT_LOAD FLAGS(7);          /* RWE */
#endif
	note PT_NOTE FLAGS(0);          /* ___ */
}
```

关于 ELF 格式中 section 和 segment 的问题: output section 可以指定自己的 load address, `PHDRS` 中也可以指定 segment 的 load address, 这二者的 load address 之间有什么关系？推理: section 是 ELF 文件中的基本存储单元，segment 是在 section 之上从 program loading 视角增加了一层(封装)抽象概念, 也就是说，section 被 map 到 segment, 通过 `readelf -l vmlinux` 可看到这种映射关系. linker script 中，output section 的定义从 virtual address 到 load address，大多数情况下都是顺序的，映射到 segment 时也应是顺序的，如果乱序映射 segment 时会怎样？本例中，`PHDRS` 命令只简单定义了几个 segment, 没有指定 load address, 且 section->segment mapping 也是顺序的，也就是说根据 `PHDRS` 中的顺序映射 section, 所以 p_paddr 由 section load address 决定.

因为 segment 是从 program loading 的角度来定义，所以被加载的 segment 在文件中的 offset(p_offset) 和它的虚拟地址(p_vaddr)最好对齐到目标架构的 page size. 这也是 ELF spec 对这两个 field 的 enforcement.

#### percpu sections & variables

linker script 中 output percpu section 的定义是：

```
PERCPU_VADDR(INTERNODE_CACHE_BYTES, 0, :percpu)
```
代码展开，列出所有相关 macro:
```
#define PERCPU_VADDR(cacheline, vaddr, phdr)				\
	__per_cpu_load = .;						\
	.data..percpu vaddr : AT(__per_cpu_load - LOAD_OFFSET) {	\
		PERCPU_INPUT(cacheline)					\
	} phdr								\
	. = __per_cpu_load + SIZEOF(.data..percpu);

#define PERCPU_INPUT(cacheline)						\
	__per_cpu_start = .;						\
	*(.data..percpu..first)						\
	. = ALIGN(PAGE_SIZE);						\
	*(.data..percpu..page_aligned)					\
	. = ALIGN(cacheline);						\
	*(.data..percpu..read_mostly)					\
	. = ALIGN(cacheline);						\
	*(.data..percpu)						\
	*(.data..percpu..shared_aligned)				\
	PERCPU_DECRYPTED_SECTION					\
	__per_cpu_end = .;

#ifdef CONFIG_AMD_MEM_ENCRYPT
#define PERCPU_DECRYPTED_SECTION					\
	. = ALIGN(PAGE_SIZE);						\
	*(.data..percpu..decrypted)					\
	. = ALIGN(PAGE_SIZE);
#else
#define PERCPU_DECRYPTED_SECTION
#endif
```
So, `PERCPU_VADDR(INTERNODE_CACHE_BYTES, 0, :percpu)` 的展开结果是：
```
__per_cpu_load = .;
.data..percpu 0 : AT(__per_cpu_load - 0xffffffff80000000) {
	__per_cpu_start = .;
    *(.data..percpu..first)
    . = ALIGN((1 << 12));
    *(.data..percpu..page_aligned)
    . = ALIGN((1 << 6));
    *(.data..percpu..read_mostly)
    . = ALIGN((1 << 6));
    *(.data..percpu)
    *(.data..percpu..shared_aligned)
    . = ALIGN((1 << 12));
    *(.data..percpu..decrypted)
    . = ALIGN((1 << 12));
    __per_cpu_end = .;
} :percpu
. = __per_cpu_load + SIZEOF(.data..percpu);
```
搜索代码发现实际 percpu variable 定义的情况是：

  1. .data..percpu..first section 中只有：`union irq_stack_union irq_stack_union`;
  2. .data..percpu..page_aligned section 中有少数变量定义: `struct gdt_page gdt_page` 和 `struct tss_struct cpu_tss_rw`;
  3. .data..percpu..read_mostly section 中有很多变量定义;
  4. .data..percpu section 中有大量变量定义;
  5. .data..percpu..shared_aligned section 不存在, 因为有 #define MODULE;
  6. .data..percpu..decrypted section 只有 AMD SME 时才有，此处视为不存在.

可以看出，percpu section 的定义有几个有趣的地方:

  1. 所有 percpu 符号的虚拟地址从 0 开始;
  2. 作为 VO image 的一部分数据，他们的 LMA(物理地址)(自然应该)与其他部分一致；
  3. 变量 __per_cpu_load 记下 percpu section 原本应有的 VMA(虚拟起始地址)，会用到。

有了上述 background knowledge, 现在可以详细分析所有 percpu 相关代码了：
```
#if defined(CONFIG_X86_64) && defined(CONFIG_SMP)
	/*
	 * percpu offsets are zero-based on SMP.  PERCPU_VADDR() changes the
	 * output PHDR, so the next output section - .init.text - should
	 * start another segment - init. 这段 comments 是 linker script PHDR 的范畴：
	 * 若上一个 output section 指定了 PHDR, 则下一个 output section 默认放在此 PHDR.
	 */
	/* 对 percpu section 的 size 做了限制。通过 readelf -s vmlinux 检查 __per_cpu_start
	 * 和 __per_cpu_end 的值发现，我的环境中 section size 只有 140k. */
	PERCPU_VADDR(INTERNODE_CACHE_BYTES, 0, :percpu)
	ASSERT(SIZEOF(.data..percpu) < CONFIG_PHYSICAL_START,
	       "per-CPU data too large - increase CONFIG_PHYSICAL_START")
#endif

	/* SKIP unrelated code */

#ifdef CONFIG_X86_32
	/* SKIP */
#else
/*
 * Per-cpu symbols which need to be offset from __per_cpu_load
 * for the boot processor.
 */
/*
 * output percpu section 的 VMA 被刻意安排于 0, LMA(物理地址) 与其他 section 一致。
 * 看过下面 head_64.S 的分析后可知: VO 的 VMA 通过页表 mapping 到实际物理地址, percpu
 * section 的虚拟地址 [0 - SIZEOF(.data..percpu)] 没有被 map, 但它原本应占据的虚拟地址
 * 空间已被 map 到它的实际物理地址。所以此时无法直接使用 percpu 变量，也就是说，无法通过
 * percpu 符号的变量名找到它的值(变量名字表示其 symbol value, 即虚拟地址).
 *
 * 这里的 trick: 为现阶段使用的 percpu variable(gdt_page, irq_stack_union) 起一个
 * 新名字, 且新名字的 symbol value 等于该 percpu variable 原本在 VO 虚拟地址空间的地址，
 * 这样就可以通过 paging 找到其实际存储位置(物理地址).
 * 在 head_64.S 中，会看到使用这两个变量的地方. Wait to see. */
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
INIT_PER_CPU(irq_stack_union);

/*
 * Build-time check on the image size:
 */
. = ASSERT((_end - _text <= KERNEL_IMAGE_SIZE),
	   "kernel image bigger than KERNEL_IMAGE_SIZE");

#ifdef CONFIG_SMP
/* output percpu section 中第一个 input section 是 .data..percpu..first section,
 * 其中只定义了 union irq_stack_union irq_stack_union; 又因为 output percpu section
 * 起始虚拟地址是 0, 所以有如下判断. */
. = ASSERT((irq_stack_union == 0),
           "irq_stack_union is not at start of per-cpu area");
#endif

#endif /* CONFIG_X86_32 */
```

### .head.text

由上一节的 linker script 可知，output section ".text" 最开头的 input section 是 .head.text, 这是内核的入口。这个 section 的内容只在两个文件中: arch/x86/kernel/head_64.S & arch/x86/kernel/head64.c, 但这两个文件的内容不仅属于 .head.text.

>问题：如何保证 .head.text 的内容里，head_64.S 在最前面？

head_64.S 是真正的入口点，来看代码:
```
/*
 * __START_KERNEL_map = 0xffffffff80000000，所以 L3_START_KERNEL = 510, 计算器可
 * 验证. 以 4-level paging 举例，一个 PGD entry 映射 512G; 一个 PUD entry(level 3)
 * 映射 1G; 一个 PMD entry 映射 2M. 而 kernel image size limit 是 512M 或 1G(KASLR),
 * 采用 2M page size. */
L3_START_KERNEL = pud_index(__START_KERNEL_map)

	/* 为什么有 2 个关于 section 的 directive? 有人帮忙找到了始作俑者: eaeae0cc985fa,
	 * 但看起来 .text 是多余的。_HEAD 引入的目的是为了保证本文件内容在 vmlinux 的最前面 */
	.text
	__HEAD
	.code64
	.globl startup_64
	/* 这是 VO 的入口点。由下面的 comments 可以看出，有些支持 64-bit boot protocol 的
	 * boot loader 可以直接解压缩 VO. */
startup_64:
	UNWIND_HINT_EMPTY /* NOT essential, 略过 */
	/*
	 * At this point the CPU runs in 64bit mode CS.L = 1 CS.D = 0,
	 * and someone has loaded an identity mapped page table
	 * for us.  These identity mapped page tables map all of the
	 * kernel pages and possibly all of memory.
	 *
	 * %rsi holds a physical pointer to real_mode_data.
	 *
	 * We come here either directly from a 64bit bootloader, or from
	 * arch/x86/boot/compressed/head_64.S.
	 *
	 * We only come here initially at boot nothing else comes here.
	 *
	 * Since we may be loaded at an address different from what we were
	 * compiled to run at we first fixup the physical addresses in our page
	 * tables and then reload them.
	 */

	/* Set up the stack for verify_cpu(), similar to initial_stack below */
	/* 因为马上要做函数调用. No need for verbiage. BUT why there? TBD.
	 * 关于 lea 指令: load effective address. Effective address 指 segment 内的
	 * offset, 因为 segment base address 是 0, 所以 effective address = linear
	 * address. 这个概念在遥远的上文解释过, 权威定义在 Intel SDM 3a, "Figure 3-5.
	 * Logical Address to Linear Address Translation". 直觉上,lea 指令应该不感知
	 * CPU 的 paging 机制，所以这里得到的地址是虚拟地址.
	 * 注意: 此刻是 identity mapped paging. */
	leaq	(__end_init_task - SIZEOF_PTREGS)(%rip), %rsp

	/* Sanitize CPU configuration */
	call verify_cpu

	/*
	 * Perform pagetable fixups. Additionally, if SME is active, encrypt
	 * the kernel and retrieve the modifier (SME encryption mask if SME
	 * is active) to be added to the initial pgdir entry that will be
	 * programmed into CR3.
	 */
	/* 此刻是 identity mapped paging, 虚拟地址 = 物理地址, lea 出 _text 的虚拟地址即
	 * VO 的物理地址. 函数返回值(在 rax 中)表示 C-bit location in page table entry */
	leaq	_text(%rip), %rdi
	pushq	%rsi
	call	__startup_64
	popq	%rsi

	/* Form the CR3 value being sure to include the CR3 modifier */
	/*
	 * 根据 x86_64 ABI 的 calling convention: _startup_64 的整数返回值(sme_me_mask)
	 * 通过 rax 传回. 不考虑 AMD SME 时，rax 值为 0; 若有 SME, 加到物理地址上作为 CR3
	 * modifier.
	 * 相减获得 early_top_pgt 的 LMA. 但是！还需加 load delta 得到它的实际物理地址! */
	addq	$(early_top_pgt - __START_KERNEL_map), %rax
	jmp 1f
ENTRY(secondary_startup_64)
	/* SKIP ...*/

1:

	/* Enable PAE mode, PGE and LA57 */
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
#ifdef CONFIG_X86_5LEVEL
	/* SKIP... */
#endif
	movq	%rcx, %cr4

	/* Setup early boot stage 4-/5-level pagetables. */
	/* __startup_64 函数已将 load delta 写入 phys_base; rax 中已写入 early_top_pgt
	 * 的 LMA, 二者相加得到 early_top_pgt 的实际物理地址, 用它更新 CR3 */
	addq	phys_base(%rip), %rax
	movq	%rax, %cr3

	/* Ensure I am executing from virtual addresses */
	/*
	 * 之前是 identity mapping page table, 根据实际物理地址建立的 page table; 现在
	 * 切换为新 page table: VMA 映射到实际物理地址。虽 CR3 已更新，但未生效, 通过更新
	 * RIP 为 VO 的 VMA 使其生效, 这样即可通过新页表找到 label 1 处的指令。
	 *
	 * 通过 objdump -d 和 readelf -r 的输出可知此处是 R_X86_64_32S 重定位类型，若无
	 * KASLR，此处 OK, 但 KASLR 情况下貌似不对? 待找答案! */
	movq	$1f, %rax
	ANNOTATE_RETPOLINE_SAFE
	jmp	*%rax
1:
	UNWIND_HINT_EMPTY

	/* Check if nx is implemented */
	/* Intel SDM 3a, chapter 4.1.4 Enumeration of Paging Features by CPUID:
	 * 返回值 EDX.NX [bit 20] = 1 表示 IA32_EFER.NXE may be set to 1, allowing
	 * PAE paging and 4-level paging to disable execute access to selected pages*/
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi

	/* Setup EFER (Extended Feature Enable Register) */
	/* Reads the contents of a 64-bit model specific register (MSR) specified
	 * in the ECX register into registers EDX:EAX. IA32_EFER 中的有效 bit 目前只
	 * 在 lower 32 bits，即存在 EAX 中. Intel SDM 4 详细介绍了 MSR. */
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_SCE, %eax	/* Enable System Call */
	btl		$20,%edi			/* No Execute supported? */
	jnc     1f					/* Jump if CF=0, i.e., if 不支持 NX */
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX, early_pmd_flags(%rip) /* set 该变量中的 NX flag */
1:	wrmsr				/* Make changes effective */

	/* Setup cr0 */
	movl	$CR0_STATE, %eax
	/* Make changes effective */
	movq	%rax, %cr0

	/* Setup a boot time stack. 函数调用(call指令)需要 stack. initial_stack 处存有
	 * 整数表示 stack 地址, 其中的符号 init_thread_union 定义在 linker script 中. */
	movq initial_stack(%rip), %rsp

	/* zero EFLAGS after setting rsp */
	pushq $0
	popfq

	/*
	 * We must switch to a new descriptor in kernel space for the GDT
	 * because soon the kernel won't have access anymore to the userspace
	 * addresses where we're currently running on. We have to do that here
	 * because in 32bit we couldn't load a 64bit linear address.
	 */
	/* early_gdt_descr 中含有 percpu 变量，相关处理参考上文 linker script 中 “percpu
	 * sections & variables” 一节。使用 vmlinux 自己的 GDT，但对注释的 "userspace
	 * address" 不解, ZO & VO 中 GDT 的 DPL 都是 0, why "userspace"? */
	lgdt	early_gdt_descr(%rip)

	/* set up data segments */
	/*
	 * 64-bit mode 下, segmentation 被部分 disable，创建了一个 flat 64-bit linear-
	 * address space. processor 默认 CS, DS, ES, SS 为 0; 但 FS, GS 依然按照传统
	 * 方式使用。commit ffb6017563aa 对下面的代码做了解释, 但目前仍没有很理解. TBD */
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es

	/*
	 * We don't really need to load %fs or %gs, but load them anyway
	 * to kill any stale realmode selectors.  This allows execution
	 * under VT hardware.
	 */
	movl %eax,%fs
	movl %eax,%gs

	/* Set up %gs.
	 *
	 * The base of %gs always points to the bottom of the irqstack
	 * union.  If the stack protector canary is enabled, it is
	 * located at %gs:40.  Note that, on SMP, the boot cpu uses
	 * init data section till per cpu areas are set up.
	 */
	/*
	 * Intel SDM 3a, chapter 3.4.4 "Segment Loading Instructions in IA-32e Mode:
	 * The hidden descriptor register fields for FS.base and GS.base are
	 * physically mapped to MSRs in order to load all address bits supported
	 * by a 64-bit implementation.
	 * wrmsr: move content of edx:eax into 64-bit MSR specified by ecx */
	/*
	 * initial_gs 处是 percpu 变量 “union irq_stack_union irq_stack_union” 原本
	 * 应有的虚拟地址(实际虚拟地址是 0), 这样才可以通过 paging 找到变量的物理地址. 注意：
	 * 此阶段, percpu 特性还没有初始化 & 使用，现在使用的 percpu 变量还是"原始数据", 这是
	 * 原注释中的 "Note that, bluhbluh" 的含义.
	 * x86 是 little-endian, 所以 initial_gs+4(%rip) 的值放在 edx(higher 32-bit).
	 * 但不知此时初始化 MSR_GS_BASE 的目的是什么?
	 *
	 * 刚开始对原注释说的 "bottom of the irqstack" 有点困惑，不确定 bottom 指一段地址的
	 * "开头" 还是 "结尾"，现在可以确定是指 "开头". 而且看过 union irq_stack_union 定义
	 * 中的注释，就可以理解 %gs:40 的含义了 */
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

	/* rsi is pointer to real mode structure with interesting info.
	   pass it to C */
	/* 注意： rsi 的值是 boot parameter 的物理地址; 此时使用的是 kernel 页表 */
	movq	%rsi, %rdi

.Ljump_to_C_code:
	/*
	 * Jump to run C code and to be on a real kernel address.
	 * Since we are running on identity-mapped space we have to jump
	 * to the full 64bit address, this is only possible as indirect
	 * jump.  In addition we need to ensure %cs is set so we make this
	 * a far return.
	 *
	 * Note: do not change to far jump indirect with 64bit offset.
	 *
	 * AMD does not support far jump indirect with 64bit offset.
	 * AMD64 Architecture Programmer's Manual, Volume 3: states only
	 *	JMP FAR mem16:16 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *	JMP FAR mem16:32 FF /5 Far jump indirect,
	 *		with the target specified by a far pointer in memory.
	 *
	 * Intel64 does support 64bit offset.
	 * Software Developer Manual Vol 2: states:
	 *	FF /5 JMP m16:16 Jump far, absolute indirect,
	 *		address given in m16:16
	 *	FF /5 JMP m16:32 Jump far, absolute indirect,
	 *		address given in m16:32.
	 *	REX.W + FF /5 JMP m16:64 Jump far, absolute indirect,
	 *		address given in m16:64.
	 */
	/*
	 * 这段代码的 style 在本文代码(ZO)分析中常见, 通过 ret 实现跳转，可能因为 AMD 64 不
	 * 支持 long jmp, 才刻意如此。
	 * initial_code 处存有 C 函数 x86_64_start_kernel 的地址, 准备跳转过去执行, 汇编
	 * 代码终于告一段落 */
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorl	%ebp, %ebp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs. GDT 中 index=2 的 entry 的 selector
	pushq	%rax		# target address in negative space
	lretq
.Lafter_lret:
END(secondary_startup_64)

#include "verify_cpu.S"

	...
	/* Both SMP bootup and ACPI suspend change these variables */
	__REFDATA
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	GLOBAL(initial_gs)
	/* 参考 linker script 一节中 percpu 相关描述 */
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
	GLOBAL(initial_stack)
	/*
	 * The SIZEOF_PTREGS gap is a convention which helps the in-kernel
	 * unwinder reliably detect the end of the stack.
	 */
	.quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
	__FINITDATA
	...

	__INITDATA

	.balign 4
GLOBAL(early_recursion_flag)
	.long 0

/* 将符号地址对齐到 PAGE_SIZE boundary */
#define NEXT_PAGE(name) \
	.balign	PAGE_SIZE; \
GLOBAL(name)

#ifdef CONFIG_PAGE_TABLE_ISOLATION
/* SKIP */
#else
	#define NEXT_PGD_PAGE(name) NEXT_PAGE(name)
	#define PTI_USER_PGD_FILL	0
#endif

/* Automate the creation of 1 to 1 mapping pmd entries */
#define PMDS(START, PERM, COUNT)			\
	i = 0 ;						\
	.rept (COUNT) ;					\
	.quad	(START) + (i << PMD_SHIFT) + (PERM) ;	\
	i = i + 1 ;					\
	.endr

	__INITDATA  /* 看起来是多余的 directive */
NEXT_PGD_PAGE(early_top_pgt)
	/* 开辟 512 x 8 = 4k 空间，在 __startup_64 函数中填充. Tip: VO 的 size limit 是
	 * KERNEL_IMAGE_SIZE, VO 的链接起始地址 __START_KERNEL_map = 0xffffffff80000000,
	 * 所以 kernel 本身的地址映射从这个虚拟地址开始. */
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0 /* 忽略 */

NEXT_PAGE(early_dynamic_pgts)
	.fill	512*EARLY_DYNAMIC_PAGE_TABLES,8,0

	.data

#if defined(CONFIG_XEN_PV) || defined(CONFIG_PVH)
/* SKIP */
#else
NEXT_PGD_PAGE(init_top_pgt)
	/* 开辟 512 x 8 = 4k 空间, 在 __startup_64 函数中填充 */
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0 /* 忽略 */
#endif

/*
 * 由上文分析知 L3_START_KERNEL = 510, 即在映射 VO 自身时，前 510 个 entry 不会被用到，
 * 初始化第 511, 512 号 entry 指向下一级页表即可. 注意: index 从 0 开始，所以相关 C 代码
 * 中是第 510, 511 号 entry.
 * 符号虚拟地址减链接起始地址，得到符号的物理地址(参考上文 “linking address of VO”)。再配上
 * 相应 page flags. 一个 PMD(level2_kernel_pgt) 映射 1G 空间，足够 cover kernel image. */
NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	/* (2^48-(2*1024*1024*1024)-((2^39)*511))/(2^30) = 510 */
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE_NOENC
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC

NEXT_PAGE(level2_kernel_pgt)
	/*
	 * 512 MB kernel mapping. We spend a full page on this pagetable
	 * anyway.
	 *
	 * The kernel code+data+bss must not be bigger than that.
	 *
	 * (NOTE: at +512MB starts the module area, see MODULES_VADDR.
	 *  If you want to increase this then increase MODULES_VADDR
	 *  too.)
	 *
	 *  This table is eventually used by the kernel during normal
	 *  runtime.  Care must be taken to clear out undesired bits
	 *  later, like _PAGE_RW or _PAGE_GLOBAL in some cases.
	 */
	/*
	 * 根据 Documentation/x86/x86_64/mm.txt: __START_KERNEL_map 映射到物理地址 0.
	 * 一个 PMD 可映射 1G 空间，足够 cover KERNEL_IMAGE_SIZE. level2_kernel_pgt 从
	 * 第一个 entry 开始映射物理地址 [0 - 512M/1G], 但实际前几个 entry 无用，因为起始
	 * 地址有个 offset = ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN) */
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

/* 开头留下 512 - 4 - 2 = 506 个 entry 应是留作映射 module 用, 待确认; then 映射 2 个
 * page table; 最后 8 M 暂不需映射，留空。 */
NEXT_PAGE(level2_fixmap_pgt)
	.fill	(512 - 4 - FIXMAP_PMD_NUM),8,0
	pgtno = 0
	.rept (FIXMAP_PMD_NUM)
	.quad level1_fixmap_pgt + (pgtno << PAGE_SHIFT) - __START_KERNEL_map \
		+ _PAGE_TABLE_NOENC;
	pgtno = pgtno + 1
	.endr
	/* 6 MB reserved space + a 2MB hole */
	.fill	4,8,0

/* 留出 2 个 page table = 8k 空间, mapping 需要. */
NEXT_PAGE(level1_fixmap_pgt)
	.rept (FIXMAP_PMD_NUM)
	.fill	512,8,0
	.endr

#undef PMDS

	.data
	.align 16
	.globl early_gdt_descr  /* GDTR 的内容 */
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	/* linker script 一节中已经介绍了 percpu section 的背景知识，这里是使用的地方。
	 * INIT_PER_CPU_VAR(gdt_page) 表示 gdt_page 的新名字，可以通过 paging 找到其
	 * 物理地址. */
	.quad	INIT_PER_CPU_VAR(gdt_page)

ENTRY(phys_base)
	/* This must match the first entry in level2_kernel_pgt */
	/*
	 * 看起来 phys_base 就是为了存储 LMA 和实际物理地址的 delta, 原注释何意? 想了半天终于
	 * 明白了！故事要从 linker script 定义的 VMA 和 LMA 说起. 上文已提到, vmlinux 占据
	 * 的地址空间被设计为：
	 *
	 *   VMA: [0xffffffff80000000 + offset -- KERNEL_IMAGE_SIZE]
	 *   LMA: [0                  + offset -- KERNEL_IMAGE_SIZE]
	 *
	 * KERNEL_IMAGE_SIZE = 512M or 1G(KASLR);
	 * offset = ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
	 *
	 * level2_kernel_pgt 可映射 1G 空间, 且是按照上述 VMA -> LMA 的关系进行映射，所以
	 * 第一个 entry 映射物理地址 0; C 函数 __startup_64 将通过将物理地址的 load_delta
	 * 加到页表中的物理地址，来修正页表映射关系，所以 phys_base 中存储的 load_delta 等于
	 * level2_kernel_pgt 中第一个 entry 中的地址! */
	.quad   0x0000000000000000
EXPORT_SYMBOL(phys_base)

```
skim-reading 函数 __startup_64 的代码后，回想 head_64.S 中开头这两个 call 指令，会引出一个问题。先看问题背景：

  1. 进入 VO 前, ZO 构建了 identity-mapped 页表，即 virtual addr = physical addr
  2. kernel 默认开启 relocatable 和 kaslr 特性，意味着 VO 将位于随机物理地址处
  3. VO 的链接起始虚拟地址是 __START_KERNEL = 0xffffffff80000000 +
     ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)

由 2. 3. 可知，进入 VO 的代码时，其虚拟地址基本不可能等于物理地址。所以，问题来了: 符号引用是如何不出错的？有如下几点原因:

  1. 以最开始的符号引用: `call verify_cpu` 和 `call __startup_64` 为例, 比对 readelf -r head_64.o
     和 objdump -d head_64.o 的输出后，发现其 relocation type 是 R_X86_64_PC32, 属于 PIC,
     即将指令中的 offset 加到 EIP 上即可得到引用符号的地址。要知道，进入 VO 时，PC(EIP) 中的
     虚拟地址 = 物理地址. So，所有 PC relative addressing 的符号引用工作正常。
  2. VO 在 head_64.S 中，根据编译时的虚拟地址和物理地址，构建了自己 page table(early_top_pgt &
     init_top_pgt), 因为 VO 的虚拟地址 != 物理地址，且有 R_X86_64_32/R_X86_64_32S/R_X86_64_64
     这样 direct addressing 的 relocation type.
  3. 如因 KASLR 导致 VO 虚拟地址随机化，ZO 中函数 handle_relocations 修正了所有 direct
     addressing 的符号引用: 通过加上虚拟地址随机化后的 delta. 又因 2. 中所述内容, 使得
     direct addressing 正常工作。
  4. 如因 relocatable 或 KASLR 导致 VO 的物理地址随机化, 则需要 fix 2. 中所述页表中的物理
     地址: 计算 linker script 中指定的物理地址和实际物理地址的 delta, 加到页表中的物理地址.
  5. 除上述修正符号引用的 case 外，还有一种需要修正的 case: 即 fixup_pointer 函数所做的.
     这个修正属于 runtime fix, 上述其他 fix 在 VO 运行之前。

针对第 5 点，以 .rela.head.text section 中第一个 R_X86_64_32S 类型重定位项为例，分析为什么需要这种符号引用的 fix. `readelf -r head64.o` 的输出：

```
Relocation section '.rela.head.text' at offset 0xd38 contains 30 entries:
...
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000003b  002d00000002 R_X86_64_PC32     0000000000000000 sme_me_mask - 4
000000000042  002b0000000b R_X86_64_32S      0000000000000000 early_top_pgt + 0
...
```
`objdump -d head64.o` 的输出：
```
Disassembly of section .head.text:

0000000000000000 <__startup_64>:
...
  2a:   48 89 fb                mov    %rdi,%rbx /* 所以 rbx 的值是入参 physaddr */
  2d:   48 89 f7                mov    %rsi,%rdi
  30:   48 89 f5                mov    %rsi,%rbp
  33:   e8 00 00 00 00          callq  38 <__startup_64+0x38>
  38:   48 8b 3d 00 00 00 00    mov    0x0(%rip),%rdi        # 3f <__startup_64+0x3f>
  3f:   4c 8d 83 00 00 00 00    lea    0x0(%rbx),%r8
...
```
可以看出：链接重定位时，将使用符号 early_top_pgt 的 symbol value(即它的虚拟地址) 填入 offset 0x42 处, 而 rbx 的值是入参 physaddr(背景: __startup_64 的入参; x86_64 ABI 的 calling convention), 二者相加得到的地址肯定找不到 early_top_pgt, 所以需要 fixup. 推理一下：找到符号 early_top_pgt 最自然的方式应该是 VO 的实际物理地址 + 符号在 VO image 中的 offset. (剧透： verification 在下方 fixup_pointer 的函数分析中)

为了对上述分析进一步确认(因为 .o 的输出没有列出 symbol value)，可以查看 `readelf -r vmlinux` 和 `objdump -d vmlinux` 的输出. 留一个上文解释过的小问题给读者: 为什么 vmlinux 中还有 relocation section?

`readelf -r vmlinux`:
```
Relocation section '.rela.text' at offset 0x10a618a8 contains 299850 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
...
ffffffff81000232  1590a0000000b R_X86_64_32S      ffffffff82798000 early_top_pgt + 0
...
```
`objdump -d vmlinux`:
```
ffffffff810001f0 <__startup_64>:
...
ffffffff8100022f:       4c 8d 83 00 80 79 82    lea    -0x7d868000(%rbx),%r8
...
```
可以确认，上述分析没有问题。一点 tips:

  - 8100022f + **3** = 81000232;(读者可以思考下 3 是怎么来的)
  - early_top_pgt symbol value =  ffffffff82798000;
  - little-endian 将 number "82798000" 表示为 "00 80 79 82";
  - R_X86_64_32S 的 S 表示 sign extended, 所以 "82798000" 可被 sign extended 为 "ffffffff82798000".

**Tip AGAIN**: 当前仍是 identity-mapped paging, 虚拟地址 = 物理地址。

继续分析 .head.text section 另一文件 head64.c(以函数出现的顺序 layout, 故与原文件 layout 不同). 第一个用到的函数是 __startup_64：
```
/* Code in __startup_64() can be relocated during execution, but the compiler
 * doesn't have to generate PC-relative relocations when accessing globals from
 * that function. Clang actually does not generate them, which leads to
 * boot-time crashes. To work around this problem, every global pointer must
 * be adjusted using fixup_pointer().
 */
/* 函数作用: fix head_64.S 中页表中的物理地址，因为页表中的物理地址是按编译时 LMA 设计的，
 * 而 VO 实际加载物理地址会因 relocatable 或 kaslr 特性而不同; 初始化页表中尚未初始化的部分 */
unsigned long __head __startup_64(unsigned long physaddr,
				  struct boot_params *bp)
{
	unsigned long vaddr, vaddr_end;
	unsigned long load_delta, *p;
	unsigned long pgtable_flags;
	pgdval_t *pgd;
	p4dval_t *p4d;
	pudval_t *pud;
	pmdval_t *pmd, pmd_entry;
	pteval_t *mask_ptr;
	bool la57;
	int i;
	unsigned int *next_pgt_ptr;

	/* 简单的判断和 fix pointer. 下方会详细分析 fixup_pointer */
	la57 = check_la57_support(physaddr);

	/* Is the address too large? */
	/* MAX_PHYSMEM_BITS 和 MAX_PHYSADDR_BITS 的区别看起来是 sparsemem 机制引入，不懂，
	 * 暂留疑。但这个判断的逻辑 self-documented. */
	if (physaddr >> MAX_PHYSMEM_BITS)
		for (;;);

	/*
	 * Compute the delta between the address I am compiled to run at
	 * and the address I am actually running at.
	 */
	/* (_text - __START_KERNEL_map) 是上文中提到的：求符号的编译时物理地址(LMA)的方式.
	 * 入参 "physaddr" 是符号 _text 的运行时地址，即实际物理地址，所以此 delta 指物理地址 */
	load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);

	/* Is the address not 2M aligned? */
	/* 2M aligned 是对 VO 加载地址的要求，即使有 relocatable 或 kaslr, 实际物理地址也
	 * 必须 2M aligned, 所以 load_delta 也将是 2M 对齐 */
	if (load_delta & ~PMD_PAGE_MASK)
		for (;;);

	/* Activate Secure Memory Encryption (SME) if supported and enabled */
	sme_enable(bp);

	/* Include the SME encryption mask in the fixup value */
	load_delta += sme_get_me_mask();

	/* Fixup the physical addresses in the page table */
	/*
	 * 入参: early_top_pgt 的编译时虚拟地址, VO 的实际物理地址。
	 * 看过 fixup_pointer 的分析可知: pgd = early_top_pgt 的实际物理地址(also 虚拟地址)。
	 *
	 * 初始化 PGD 中的对应地址 __START_KERNEL_map 的 entry: 找到下一级页表的物理地址.
	 * 以 level3_kernel_pgt 为例将等式展开：
	 *
	 * *p = level3_kernel_pgt + _PAGE_TABLE_NOENC - __START_KERNEL_map + load_delta
	 *    = level3_kernel_pgt - __START_KERNEL_map + load_delta + _PAGE_TABLE_NOENC
	 *      (符号的链接虚拟地址)   (VO的链接起始虚拟地址)
	 *
	 * 前二者相减得到 level3_kernel_pgt 链接时的 LMA(物理地址), 上文中已解释过这类表达式。
	 * 加上 load_delta 得到符号的实际物理地址; _PAGE_TABLE_NOENC 是 flags.
	 * 明显, __START_KERNEL_map(0xffffffff80000000) 对应 PGD 的最后一个 entry. */
	pgd = fixup_pointer(&early_top_pgt, physaddr);
	p = pgd + pgd_index(__START_KERNEL_map);
	if (la57)
		*p = (unsigned long)level4_kernel_pgt;
	else
		*p = (unsigned long)level3_kernel_pgt;
	*p += _PAGE_TABLE_NOENC - __START_KERNEL_map + load_delta;

	if (la57) {
		p4d = fixup_pointer(&level4_kernel_pgt, physaddr);
		p4d[511] += load_delta;
	}

	/* pud = level3_kernel_pgt 的实际物理地址(also 虚拟地址). 需理解 head_64.S
	 * 中符号 level3_kernel_pgt 的定义，才能明白为什么是 pud[510], pud[511]. */
	pud = fixup_pointer(&level3_kernel_pgt, physaddr);
	pud[510] += load_delta;
	pud[511] += load_delta;

	/* pmd = level2_fixmap_pgt 的实际物理地址(also 虚拟地址). 同样需理解 level2_fixmap_pgt
	 * 在 head_64.S 中的定义. 更新 506, 507 两个 entry. */
	pmd = fixup_pointer(level2_fixmap_pgt, physaddr);
	for (i = FIXMAP_PMD_TOP; i > FIXMAP_PMD_TOP - FIXMAP_PMD_NUM; i--)
		pmd[i] += load_delta;

	/*
	 * Set up the identity mapping for the switchover.  These
	 * entries should *NOT* have the global bit set!  This also
	 * creates a bunch of nonsense entries but that is fine --
	 * it avoids problems around wraparound.
	 */
	/*
	 * next_pgt_ptr = next_early_pgt 的实际物理地址(also 虚拟地址). 后者定义在本文件
	 * 开头, 是 int 数据，为何这样使用? Lucky, 直接 blame 代码就找到 187e91fe5e91:
	 * 为了兼容 Clang 编译器.
	 *
	 * 代码分析至此，发现上面忽略了一个小细节: C 中如何引用 assembly label? 之前已经了解
	 * C 中如何引用 linker script 中的符号，这两种情况有细微的不同，参考：
	 * https://stackoverflow.com/questions/8045108/use-label-in-assembly-from-c
	 *
	 * early_dynamic_pgts 被声明为二维指针数组，所以 pud = early_dynamic_pgts 的
	 * 实际物理地址, pmd = early_dynamic_pgts 实际物理地址 + 4k. */
	next_pgt_ptr = fixup_pointer(&next_early_pgt, physaddr);
	pud = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
	pmd = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);

	pgtable_flags = _KERNPG_TABLE_NOENC + sme_get_me_mask();

	if (la57) {
		/* SKIP... */
	} else {
		/* pgd 中的 512 个 entry, 目前只用了最后一个 entry. 如注释所说(目前不理解)，
		 * 再做一个 identity mapping. 只需一个 PGD entry(PUD), [i + 1] 是何意?
		 * 答: 应是为 AMD SME. 参考 AMD64 programmer manual volume 2:
		 * 7.10.8 Encrypt-in-Place. */
		i = (physaddr >> PGDIR_SHIFT) % PTRS_PER_PGD;
		pgd[i + 0] = (pgdval_t)pud + pgtable_flags;
		pgd[i + 1] = (pgdval_t)pud + pgtable_flags;
	}

	/* 同样，只需一个 PUD entry(PMD), [i + 1] 是何意? 同上: AMD SME, Encrypt-in-Place */
	i = (physaddr >> PUD_SHIFT) % PTRS_PER_PUD;
	pud[i + 0] = (pudval_t)pmd + pgtable_flags;
	pud[i + 1] = (pudval_t)pmd + pgtable_flags;

	/* 初始化: 用 VO 实际物理地址对应的第一个 PMD entry 的值作 base value */
	pmd_entry = __PAGE_KERNEL_LARGE_EXEC & ~_PAGE_GLOBAL;
	/* Filter out unsupported __PAGE_KERNEL_* bits: */
	mask_ptr = fixup_pointer(&__supported_pte_mask, physaddr);
	pmd_entry &= *mask_ptr;
	pmd_entry += sme_get_me_mask();
	pmd_entry +=  physaddr;

	/* 其余 entry 的值只需更新 physical address */
	for (i = 0; i < DIV_ROUND_UP(_end - _text, PMD_SIZE); i++) {
		/* 找到 physaddr 对应 PMD 中的第一个 index, 不断 + i 直到映射 whole VO image
		 * size; 用 PMD_SIZE 不断更新 mapped physical address. */
		int idx = i + (physaddr >> PMD_SHIFT) % PTRS_PER_PMD;
		pmd[idx] = pmd_entry + i * PMD_SIZE;
	}

	/*
	 * Fixup the kernel text+data virtual addresses. Note that
	 * we might write invalid pmds, when the kernel is relocated
	 * cleanup_highmap() fixes this up along with the mappings
	 * beyond _end.
	 */
	/* 为何这块 fix 页表中物理地址的代码与上面相隔那么远? 实验证明: 将此 code block 搬
	 * 到上面，kernel 可正常启动。但 Maintainer 很可能认为这种修改 pointless.
	 * 特意判断 _PAGE_PRESENT, 因为此 PMD 中的 entry 可能用了一半，也可能全部用了, 若
	 * 只使用了一半, 则没必要 loop all entries. */
	pmd = fixup_pointer(level2_kernel_pgt, physaddr);
	for (i = 0; i < PTRS_PER_PMD; i++) {
		if (pmd[i] & _PAGE_PRESENT)
			pmd[i] += load_delta;
	}

	/*
	 * Fixup phys_base - remove the memory encryption mask to obtain
	 * the true physical address.
	 */
	/* phys_base 定义在 head_64.S，default to 0, 这里只加上 delta 明显不够, 在本函数
	 * 结束后, head_64.S 有对其继续处理. 但原注释明显 NOT enough! */
	*fixup_long(&phys_base, physaddr) += load_delta - sme_get_me_mask();

	/* 下面的代码都是 AMD SME 相关，SKIP... */
	/* Encrypt the kernel and related (if SME is active) */
	sme_encrypt_kernel(bp);

	/*
	 * Clear the memory encryption mask from the .bss..decrypted section.
	 * The bss section will be memset to zero later in the initialization so
	 * there is no need to zero it after changing the memory encryption
	 * attribute.
	 */
	if (mem_encrypt_active()) {
		vaddr = (unsigned long)__start_bss_decrypted;
		vaddr_end = (unsigned long)__end_bss_decrypted;
		for (; vaddr < vaddr_end; vaddr += PMD_SIZE) {
			i = pmd_index(vaddr);
			pmd[i] -= sme_get_me_mask();
		}
	}

	/*
	 * Return the SME encryption mask (if SME is active) to be used as a
	 * modifier for the initial pgdir entry programmed into CR3.
	 */
	/* Memory Encryption mask: C-bit location in page table entry. mask 形式 */
	return sme_get_me_mask();
}

static void __head *fixup_pointer(void *ptr, unsigned long physaddr)
{
	/* Verify 我们的推理的时候到了： ptr(符号的编译地址) 减去 _text(VO 的编译起始地址),
	 * 就是推理中所说的符号的 offset, 再加到 VO 的物理地址上，即得到符号的物理地址，或曰
	 * 运行地址. identity-mapped paging, 物理地址 = 虚拟地址。*/
	return ptr - (void *)_text + (void *)physaddr;
}


```

## APPENDIX

### 常见汇编指令快速参考

#### CMP

比较源操作数和目的操作数，根据结果设置 EFLAGS 寄存器的 status flags。所谓比较是指： 目的操作数 - 源操作数(结果被丢弃)。Jcc 指令常常跟在 CMP 后，根据刚刚 CMP 的结果进行条件跳转。举例(AT&T汇编语法，第一个操作数是源操作数)：

	cmp a, b
	jg label

意味着：jump to label if and only if b > a

#### TEST

对 源操作数 和 目的操作数 做逻辑与，根据结果设置 EFLAGS 寄存器的 SF，ZF，PF 状态位，不保存逻辑与后的结果。

#### BT - Bit Test
Selects the bit in a bit string at the bit-position designated by the bit offset and stores the value of the bit in the CF flag. 所以通常后面跟指令 Jnc(Jump short if not carry (CF=0)) 或 Jc(Jump short if carry (CF=1)).

#### BTS - Bit Test and Set
Selects the bit in a bit string at the bit-position designated by the bit offset operand, stores the value of the bit in the CF flag, and sets the selected bit in the bit string to 1.

#### RDMSR
Reads the contents of a 64-bit model specific register (MSR) specified in the ECX register into registers EDX:EAX. The EDX register is loaded with the high-order 32 bits of the MSR and the EAX register is loaded with the low-order 32 bits.

#### WRMSR
Writes the contents of registers EDX:EAX into the 64-bit model specific register (MSR) specified in the ECX register. The contents of the EDX register are copied to high-order 32 bits of the selected MSR and the contents of the EAX register are copied to low-order 32 bits of the MSR.