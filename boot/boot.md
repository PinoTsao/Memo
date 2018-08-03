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

## GRUB2 启动流程

本节以只有一块硬盘的 PC 为例，基于 grub2 的代码分析 grub 的工作流程。这篇材料：[grub2-booting-process](https://www.slideshare.net/MikeWang45/grub2-booting-process) 可以作为很好的参考。

BIOS 的最后工作是将硬盘 MBR(也叫 boot sector) 中的内容加载到地址 0000:7C00，并跳转过去继续执行。为什么将 MBR 加载到这个比 32kb 小 1024 byte 的地址？ 这里有[一篇科普](http://www.ruanyifeng.com/blog/2015/09/0x7c00.html)。使用 MBR 这个术语是为了和 VBR 区分，Master boot record 是指整个硬盘的第一个 sector，Volume boot record 是指一个分区的第一个 sector。

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
         /* 上面的注释说：如果 grub 被安装在硬盘上，dl 唯一的有效值是 0x80，因为这是唯一可能的 boot drive.
          * 如果是安装在软盘上的，这一段 do nothing. */
		.org GRUB_BOOT_MACHINE_DRIVE_CHECK
	boot_drive_check:
		/* 当 grub 安装在 HDD 时，某些有问题的 BIOS 会错误的设置 DL 寄存器，所以 jmp 指令
		 * 才可能被 overwrite，进入下一条指令进行相关判断。如果是安装在软盘时，则不存在这个问题，
		 * 直接跳转到 3:, 进行软盘相关的判断。可以参考上面的文章 <BIOS to MBR interface> */
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


真正的代码开始于 real_start：通过调用 INT 0x13 判断磁盘是否支持 LBA 访问模式，否则使用传统的 CHS 模式。以 LBA 为例，继续调用 INT 0x13，从 core.img 的第一个 sector 的起始位置读入一个 sector(代码注释是："the blocks") 到地址为 *GRUB_BOOT_MACHINE_BUFFER_SEG*(0x7000):0 的 buffer 中，关于这个 INT 0x13 调用的详细解释参考[wikipedia](https://en.wikipedia.org/wiki/INT_13H)或者[这篇](http://www.ctyme.com/intr/rb-0708.htm)。然后调用函数 *copy_buffer*，从源地址 *GRUB_BOOT_MACHINE_BUFFER_SEG*:0 拷贝 512 bytes 到 0:*GRUB_BOOT_MACHINE_KERNEL_ADDR*，也即从 0x7000:0 拷贝到 0:0x8000，然后 jmp 到 *GRUB_BOOT_MACHINE_KERNEL_ADDR*。

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

一眼看上去感觉乱乱的，实际上应该是为了合理利用 ax 寄存器的值才使的顺序上变这个样子。

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
		/* 将 core.img 剩余部分内容的起始地址(sector No. in LBA mode)放入寄存器 ebx 和 ecx，
		 * 下面调用 INT 13 时候会用到 */
		movl	(%di), %ebx
		movl	4(%di), %ecx

		/* the maximum is limited to 0x7f because of Phoenix EDD */
		/* Phoenix EDD 的介绍可以参考 INT 13 的 wikepedia 页面：
		 * To support even larger addressing modes, an interface known as
		 * INT 13h Extensions was introduced by Western Digital and Phoenix Technologies
		 * as part of BIOS Enhanced Disk Drive Services (EDD).
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
		/* 寄存器 di 保存着 firstlist 的地址，这个地址指向待读取的 sector 的地址，也即 sector 号，
		 * 每次读取 sector 后，下次读取的地址自然也要更新，新地址 = 老地址 + 上次读取的 sector 数 */
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
		/* 10(%di) 处保存着 buffer 数据被 cp 到目的段的段基址： GRUB_BOOT_MACHINE_KERNEL_SEG + 0x20，
		 * 即 0x820。这里使用的策略是：每次 cp 使用的目的地址是 10(%di): 0，即每次 offset 都是 0，
		 * 那么每次 cp 后只更新段基址 10(%di) 即可形成下次 cp 的地址 */
		movw	10(%di), %es	/* load destination segment */

		/* restore %ax */
		/* ax 中保存着刚刚 INT 13 读取的磁盘 sector 数 */
		popw	%ax

		/* determine the next possible destination address (presuming 512 byte sectors!) */
		/* 为啥表示 sector 数的 %ax 左移 5 bit？每次从 buffer 中 cp %ax 个 sector 内容到
		 * 目的地址 10(%di):0，所以每次 cp 后，目的地址 = 目的地址 + %ax * 512，即 %ax 需要左移 9 bit。
		 * 因为使用的策略是仅更新段基址，所以左移 5 bit 后的值加到 10(%di) 处的段基址上，又因为
		 *  logical address 转换 linear address 时，段基址还要左移 4 bit，所以其实一共左移 9 bit，
		 *  也就达到了 %ax * 512(2^9) 的目的。 */
		shlw	$5, %ax		/* shift %ax five bits to the left */
		addw	%ax, 10(%di)	/* add the corrected value to the destination
							   address for next time */

		/* save addressing regs */
		pusha
		pushw	%ds

		/* get the copy length */
		/* 上面左移了 5 bit，这里又左移了 3 bit，共 8 bit，也即 %ax = %ax * 2^8 = %ax * 256。
		 * 因为一个 sector 是 512 byte，且下面使用的是 movsw 指令，每次 mov 一个 word(2 bytes)，
		 * 所以 cp 一个 sector 只需要 512 / 2 = 256(2^8) 次即可。所以，一共移动 %ax << 8 次即可 */
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
		/* 如果检查发现这个 blocklist 中的 sector 都读取完了，那么查看是否还有 blocklist 需要读取，
		 * 如果还有 blocklist 的话，上面已经解释过，它将紧挨着 diskboot.S 文件底部的 firstlist,
		 * 所以减去 12 即可 */
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

%edi 被赋值为 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR(0x100000=1M)，然后被 push 到 stack 上保存，调用 _LzmaDcodeA 函数后又 popl %esi，最后 jmp *%esi，也即 jmp 到 GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR。这个跳转指令使用了不常见的 *%esi 形式，语法解释在9.15.3.1 of `info as`：AT&T absolute (as opposed to PC relative) jump/call operands are prefixed by '*'。这里又引入一个知识点：绝对跳转(absolute jump) vs 相对跳转(relative jump)，参考 intel 指令手册的 JMP 指令。

### kernel.img

看一下构建 kernel.img 的 Makefile 内容，在 grub-core/Makefile.core.am:

	if COND_i386_pc
	platform_PROGRAMS += kernel.exec
	kernel_exec_SOURCES  = kern/i386/pc/startup.S
	kernel_exec_SOURCES += kern/i386/pc/init.c kern/i386/pc/mmap.c term/i386/pc/console.c kern/	i386/dl.c
						 kern/i386/tsc.c kern/i386/tsc_pit.c kern/compiler-rt.c kern/mm.c kern/time.c kern/generic/millisleep.c
						 kern/command.c kern/corecmd.c kern/device.c kern/disk.c kern/dl.c kern/env.c 	kern/err.c kern/file.c
						 kern/fs.c kern/list.c kern/main.c kern/misc.c kern/parser.c kern/partition.c kern/rescue_parser.c
						 kern/rescue_reader.c kern/term.c
	...
	kernel_exec_LDFLAGS  = $(AM_LDFLAGS) $(LDFLAGS_KERNEL) $(TARGET_IMG_LDFLAGS) $(TARGET_IMG_BASE_LDOPT),0x9000
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

		/* 窍门：符号 cont 的地址是基于 0x9000，加上当前的 location counter。刚刚已经把 kernel.img
		 * copy 回它应该在的地址，这里通过绝对跳转，跳回符号 cont 的地址继续执行。也就是说，绝对跳转后
		 * 执行的代码是取自 0x9000 的 kernel.img，不是 0x100000 的( buffer 中的)kernel.img */
		movl	$LOCAL (cont), %esi
		jmp	*%esi
	LOCAL(cont):

这里有个不常见的符号: _edata，可以通过 `man etext/edata/end/` 来了解，ld 的默认链接脚本会定义了这几个符号。那么如何查看 ld 的 default linker script 呢？ 使用 `ld --verbose` 可以查看默认链接脚本的完整内容。继续看后面的代码

	LOCAL(cont):
	...

		/* clean out the bss */
		/* 跳转回来后第一件事是为 bss section 手动清零，BSS_START_SYMBOL 的值默认是 "__bss_start"，
		 * END_SYMBOL 默认值是 "end"，都在 ld 的默认链接脚本中定义。*/
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
	  /* 如果 module info data 中有 OBJ_TYPE_CONFIG 类型的对性，此函数才有用。不过一般都没有 */
	  grub_load_config ();
	  ...
	  /* 函数顾名思义，kernel.img 中有专门的数据结构 grub_symtab 来保存一些符号，目前看起来在 module 重定位
	   * 时会用到。这个函数将 kernel.img 自身的一些符号信息先注册到 grub_symtab 数据结构中。待确认和详细分析 */
	  grub_register_exported_symbols();
	  ...
	  /* 此函数将所有 buffer 中的 module 加载并链接起来，使其可用，内容复杂，在下面有单独分析 */
	  grub_load_modules ();
	  ...
	  /* Reclaim space used for modules.
	   * buffer 中的 module 使用完了，buffer 空间就没用了，回收 */
	  reclaim_module_space ();
	  ...
	  /* 注册一些核心命令，这些命令是由 kernel.img 自己提供。更多的命令还是由各种 module 提供 */
	  grub_register_core_commands ();
	  ...
	  /* 加载 grub 目录中的 normal 模块，其加载过程和函数 grub_load_modules 中一样。加载后则执行
	   * "normal"命令。normal 模块会加载 linux kernel, initramfs */
	  grub_load_normal_mode ();
	}

第一个函数 grub_machine_init 处理的事情比较多，拿出来单独分析：

	void grub_machine_init (void)
	{
	  /* 这个函数不需要研究，它是针对 VIA 的芯片做单独的 workaround */
	  grub_via_workaround_init ();

	  /* 上文已说过，解压缩的临时 buffer 地址位于地址 1M(0x100000)处，压缩数据前端是 kernel.img，
	   * 后面就是各种 module，因为 bss section 是不占据文件空间的，所以 _edata - _start 就是 kernel.img
	   * 的有效 size，module 又是紧挨着 kernel.img，所以 grub module 的起始地址计算如下 */
	  grub_modbase = GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR + (_edata - _start);
	  ...

	  /* 重点是下面关于内存的初始化。其实比较简单，函数 grub_machine_mmap_iterate 通过 e820 中断获得
	   * 内存信息(addr, len, type)，然后将信息交给函数 mmap_iterate_hook 处理，过滤掉地址小于1M 且
	   * 大于4G的，将类型是 GRUB_MEMORY_AVAILABLE 的内存区域信息保存到数组 mem_regions[] 中 */
	  grub_machine_mmap_iterate (mmap_iterate_hook, NULL);
	  /* 通过此函数整理数组 mem_regions[]：按地址从小到大排序，如果2快内存区域有重叠，则合二为一 */
	  compact_mem_regions ();

	  /* 获得解压缩 buffer 中的 module 的结束地址，要看明白此函数，需要了解打包的 core.img 的格式，
	   * 在“安装 GRUB”一节中有图示。然后通过 grub_mm_init_region 做 grub 的内存管理功能的初始化。
	   * 因为不是重点，所以对此函数不做详细。看起来由两级数据结构来管理：grub_mm_region_t &
	   * grub_mm_header_t。初始化后，grub 中所有的 malloc 类操作都是在操作 grub_mm_base 这个
	   * 数据结构了 */
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

grub_load_modules 函数比较复杂，下面拿出来单独分析。核心内容是遍历 buffer 中所有类型为 OBJ_TYPE_ELF 的 module，将其代码和数据加载到一块分配的内存中并进行重定位，然后执行 module 的初始化函数。

>grub_load_modules --遍历module--> grub_dl_load_core --> **grub_dl_load_core_noinit**
                                                    \
>                                                    --> grub_dl_init

重点都在函数 grub_dl_load_core_noinit 中了

	/* 入参 addr 是 module 在 buffer 中的地址 */
	grub_dl_t grub_dl_load_core_noinit (void *addr, grub_size_t size)
	{
	  Elf_Ehdr *e;/* module 是 ELF 文件 */
	  grub_dl_t mod;/* 用来描述一个 module 的总结构体 */
	  ...
	  /* ELF header 的常规检查，很常规，不做分析 */
	  grub_dl_check_header(e, size);
	  ...

	  /* 重要的处理就在下面这些函数中。总的来说就是对 ELF 文件的 section 做操作，要看懂下面的函数，
	   * 需要读一下 `man elf`。只对重点函数详细介绍。grub_dl_check_license 和
	   * grub_dl_resolve_name 的内容很简单，略过；grub_dl_resolve_dependencies 也比较简单，
	   * 找一下是否有 module 依赖关系的 section，如果有则先加载依赖的 module；其余三个函数工作量比较大，
	   * 下面重点分析。*/
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
	  /* 遍历所有的 section 获得 total size, 和 total align。total align 就是所有 alignment 中
	   * 最大的那个，total size 是将所有 section size 对其到 alignment 后的和 */
	  for (i = 0, s = (const Elf_Shdr *)((const char *) e + e->e_shoff);
		   i < e->e_shnum;
      	   i++, s = (const Elf_Shdr *)((const char *) s + e->e_shentsize))
      {
        tsize = ALIGN_UP (tsize, s->sh_addralign) + s->sh_size;
        if (talign < s->sh_addralign)
		talign = s->sh_addralign;
	  }
	  /* 按 total size 和 align 分配内存 */
	  mod->base = grub_memalign (talign, tsize);
	  mod->sz = tsize;
	  ptr = mod->base;

	  /* 再一次遍历 module 的所有 section */
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

			/* 然后copy到目的内存中 */
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

	/* 此函数对 buffer 中的 symbol table section 解析并原地修改 */
	static grub_err_t grub_dl_resolve_symbols (grub_dl_t mod, Elf_Ehdr *e)
	{
	  /* 遍历 sections 找到符号表 section */
	  for (i = 0, s = (Elf_Shdr *) ((char *) e + e->e_shoff);
           i < e->e_shnum;
       	   i++, s = (Elf_Shdr *) ((char *) s + e->e_shentsize))
    	if (s->sh_type == SHT_SYMTAB)
    	  break;

	  mod->symtab = (Elf_Sym *) ((char *) e + s->sh_offset); /* 记下符号表在 buffer 中的地址 */
	  mod->symsize = s->sh_entsize;/* 记下符号表 entry 的 size */
	  ...
	  /* sh_link 在 man elf 并没有详细解释，详细解释参考：
	  * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html#chapter6-47976
	  * 对于符号表 section 来说，sh_link 是其对应 string table 的 section index */
	  s = (Elf_Shdr *) ((char *) e + e->e_shoff + e->e_shentsize * s->sh_link);
	  str = (char *) e + s->sh_offset;/* 拿到 string table 的地址*/

	  /* 开始循环解析符号表中的每一个 entry */
	  for (i = 0;
	       i < size / entsize;
	       i++, sym = (Elf_Sym *) ((char *) sym + entsize))
	  {
	    switch (type)
	    {
		  case STT_NOTYPE:
		  case STT_OBJECT:

		  /* Resolve a global symbol.  */
		  /* https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblh/index.html, table 7-11 */
		  /* 如果符号有名字，且 st_shndx 是 SHN_UNDEF(0)，说明是定义在别处(module or kernel.img)的符号 	*/
		  if (sym->st_name != 0 && sym->st_shndx == 0)
		  {
			/* 这个未定义的 symbol 必须已注册在内部的 grub_symtable 中，否则错误返回 */
			grub_symbol_t nsym = grub_dl_resolve_symbol (name);
	        if (! nsym)
			  return grub_error (GRUB_ERR_BAD_MODULE,
					   N_("symbol `%s' not found"), name);

			  /* 找到的话，则把符号的地址赋值给 st_value；并更新 symbol entry 的 st_info field */
		      sym->st_value = (Elf_Addr) nsym->addr;
		      if (nsym->isfunc)
				sym->st_info = ELF_ST_INFO (bind, STT_FUNC);
		  }
		  else
		  {
		    /* 如果这个 symbal 在本 module 中定义 */
		    /* 关于 symbal table entry 中的 st_value 的定义，请参考：
		     * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/	index.html#chapter6-35166
		     * 本例中，st_value 是符号在所在 section 中的 offset，加上 section address，构成 st_value */
		    sym->st_value += (Elf_Addr) grub_dl_get_section_addr (mod,
								    sym->st_shndx);
			/* 如果不是 local 的符号，则注册到 grub_symtable 中 */
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

	/* 遍历 buffer 中 module 的重定位 section，对cp到已分配内存中代码/数据进行重定位 */
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
		  /* 对于 sh_info 的详细解释，还是参考这里：
		   * https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html#chapter6-47976
		   * 对于类型是 SHT_REL/SHT_RELA 的 section 来说，sh_info 指的是：
		   * The section header index of the section to which the relocation applies.*/
		  for (seg = mod->segment; seg; seg = seg->next)
		    if (seg->section == s->sh_info)
		      break;

		  /* 找到了要进行重定位的 section，调用函数进行重定位。注意，这个 section 的地址是
		   * 之前 grub_dl_load_segments 后分配的，而不是解压缩 buffer 中的。 */
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

	/* 重定位函数，不同的ABI有不同的实现，x86下分 i386 和 x86_64 两种，我们以 i386 为例进行分析 */
	grub_err_t grub_arch_dl_relocate_symbols (grub_dl_t mod, void *ehdr, Elf_Shdr *s, grub_dl_segment_t seg)
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

		/* 根据重定位类型手动进行重定位。在之前的函数 grub_dl_resolve_symbols 中，已将 buffer 中
		 * 符号表 entry 的 st_value 修改为符号的地址。下面的重定位也很容易理解，不赘述。 */
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

Ok，终于介绍完了 grub_load_modules 函数，回头继续看 grub_main，倒数第二个函数是 grub_load_normal_mode，也即加载 normal 模块并执行 normal 命令，关于 grub module 可以参考阅读"GRUB modules 简介"一节。normal module/command 的介绍参考[官方文档](https://www.gnu.org/software/grub/manual/grub/grub.html#normal)。

作为 kernel.img/grub_main 过程重要的一部分，我们仍然捡重点代码分析 normal 命令的过程：

	static grub_err_t grub_cmd_normal (struct grub_command *cmd __attribute__ ((unused)), int argc, char *argv[])
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
	  /* 读取 grub 目录中各种文件，读取 command.lst，fs.lst，crypto.lst，terminal.lst 保存到内部数据结构中；
	   * 读取 grub.cfg 保存到 grub_menu_t 结构中，并执行其中的命令(应该是 grub.cfg 上方，menuentry 以外的那些命令)，
	   * grub_menu_t 结构包括了显示 grub menu 所需的数据。展示 menu 菜单，获得用户选择的 menu entry
	   * 或者 timeout 后的 default entry，执行这个 entry 中的各种命令用来启动 OS */
	  grub_normal_execute (config, 0, 0);
	  /* 正常情况下，下面这个函数不会走到？因为上面的函数已经成功启动 OS 了，只有在无法启动 OS
	   * 的异常情况下，grub_normal_execute 才返回？ */
	  grub_cmdline_run (0, 1);
	  ...
	}

展示 grub menu，获得 menu entry 并执行这个 entry 的 callchain 长这样：

>grub_normal_execute
 	--> grub_show_menu
 	    --> show_menu
 	        --> boot_entry = run_menu (menu, nested, &auto_boot);
 	        --> e = grub_menu_get_entry (menu, boot_entry);
 	        --> grub_menu_execute_with_fallback (menu, e, autobooted, &execution_callback,0);
 	        --> grub_menu_execute_entry /* Run a menu entry */
 	        	--> grub_script_execute_new_scope
 	        		--> grub_script_execute_sourcecode /* 看起来像在逐行解析 menu entry 的内容，并执行对应的命令，比如 linux16, initrd16 等等 */
 	        	--> grub_command_execute ("boot", 0, 0) /* 执行完 entry 中的各种命令，可以启动 OS 了 */

上述过程有待详细分析。分析到这一步，我们所关心的 grub 的工作流程，就剩下 menu entry 中用于加载 linux kernel 和 initramfs 的两条命令比较重要，将单独作为一节进行分析，因为它涉及 Linux kernel 的内容，将在 “normal 模块加载 linux kernel & initramfs”一节中进行分析。

#### GRUB modules 简介
Module 的概念在 grub2 中引入，有两篇文章可以作为科普：

1. [Writing GRUB Modules](https://wiki.osdev.org/Writing_GRUB_Modules)
2. [grub2-modules](http://blog.fpmurphy.com/2010/06/grub2-modules.html?output=pdf)

我们从代码的角度简单分析 module 的实现框架。每一个 module 都需要 initialization 和 finalization 函数，分别由宏 GRUB_MOD_INIT 和 GRUB_MOD_FINI 辅助完成，他们的定义在 include/grub/dl.h 中：

	/* 为了简洁，直接列出在 i386-pc 平台上该宏的定义 */
	#define GRUB_MOD_INIT(name)	\
	static void grub_mod_init (grub_dl_t mod __attribute__ ((unused))) __attribute__ 	((used)); \
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

### normal 模块加载 linux kernel & initramfs

grub-mkconfig 生成 grub.cfg 时，会根据实际环境在 menuentry 中使用 linux16/initrd16 或者 linux/initrd 命令，究竟如何决定，代码细节尚未分析，在此也暂时略过。现在只需要知道，他们分别对应了 16-bit/32-bit 的 linux/x86 boot protocal 即可，boot protocal 在 linux kernel 的 Documentation/x86/boot.txt 中有详细介绍。本文将以 16-bit boot protocal 为例进行代码分析。



## 安装 GRUB

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
	/* 将 kernel.img 和 modules 打包在一起并压缩。压缩后的数据由 core_img 表示，大小是 core_size */
	compress_kernel (image_target, kernel_img, layout.kernel_size + total_module_size,
		   &core_img, &core_size, comp);

打包后，压缩前的数据长这样：

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
	   * len 字段表示 core.img 中除 diskboot.img 以外的 size(sector 为单位)。只写入 size，
	   * 起始地址在安装的时候才知道 */
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

但直接搜 grub_util_bios_setup 也找不到，原来定义在 util/setup. 中：

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
	/* 省略一段 root device 分析，因为目前不确定它的含义。目前的理解是：一台 PC 上可能由很多块
	 * 磁盘，安装了 grub 或操作系统那一块才叫 root device? 那么一般 PC 只有一块硬盘，它就是 root device
	 * 艰难的看了这段代码，理解过程如下，从 /proc/self/mountinfo 中读取信息，跟入参 dir: /boot/grub/i386-pc
	 * 比对，找到 mountinfo 中 mount point 是 /boot 的 mount source(参考 man proce)，
	 * 最后确定 root device, 并设置到环境变量。我的测试中显示 root = hostdisk//dev/sda,msdos1 */
	grub_util_info ("setting the root device to `%s'", root);
	if (grub_env_set ("root", root) != GRUB_ERR_NONE)
		grub_util_error ("%s", grub_errmsg);

	#ifdef GRUB_SETUP_BIOS
	  {
		/* Read the original sector from the disk.  */
		/* 读取当前 boot sector 中的内容，作用是：1. copy 当前 boot sector 可能存在的 BPB 数据;
		 * 2. 修改指令适配有问题的 BIOS; 3. copy 分区表 */
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
		/* 上面的注释把 bug 解释的很清楚，关于 boot drive number 的问题在 boot.img
		 * 一节已有解释。看起来一般情况下都会改写 boot.img 中 boot_drive_check 处的 jmp 指令 */
		if (!allow_floppy && !grub_util_biosdisk_is_floppy (dest_dev->disk))
		{
			/* Replace the jmp (2 bytes) with double nop's.  */
			boot_drive_check[0] = 0x90;
			boot_drive_check[1] = 0x90;
		}
		...
		/* 这段代码用来获得 core image 将要安装到磁盘的位置(sector号)，存放在数组 sectors */
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
		/* 在此函数中更新 diskboot.img 中的 blocklist 数据，细节掠过 */
		save_blocklists (sectors[i] + grub_partition_get_start (ctx.container),
		       0, GRUB_DISK_SECTOR_SIZE, &bl);

		/* 已经确定了 core.img 将要安装的 sector 地址，也要更新 boot.img 中的相关字段，使得
		 * boot.img 可以找到 core.img 的第一个 sector 的内容，也即 diskboot.img */
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
	/* grub_install_source_directory 是上文中说的 image directory，target 在我们的例子中是 "i386-pc" */
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
      /* 对于 i386-pc 平台来说，默认访问磁盘的方式是通过 BIOS INT13，所以 diskmodule
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

	/* 对于非 efi 来说，直接来到下面的函数，将 image directory 下的内容 copy 到 boot directory。
	 * copy 的内容包括：image directory 下所有的 .mod 文件；{"efiemu32.o", * "efiemu64.o",
	 * "moddep.lst", "command.lst", "fs.lst", "partmap.lst", "parttool.lst", "video.lst",
	 * "crypto.lst", "terminal.lst", "modinfo.sh"} 等 12 个文件； image directory 下的 po 目录，
	 * 即 locale 文件，在我们的测试环境中没有 po 目录； /usr/lib/grub/theme(我们的测试下环境)
	 * 下的 theme 文件; /usr/lib/grub/ 下的 fonts 文件(.pf2) */
	grub_install_copy_files (grub_install_source_directory,
							 grubdir, platform);

	/* 跨过一堆磁盘/文件系统检测相关的代码，终于来待生成 core.img 的代码 */
	/* 不得不说， grub 的代码写的有点乱，这里的 platdir 指 boot directory，即 /boot/grub/i386-pc，
	 * 所以 imgfile 指 /boot/grub/i386-pc/core.img */
	char *imgfile = grub_util_path_concat (2, platdir, core_name);
	char *prefix = xasprintf ("%s%s", prefix_drive ? : "", relative_grubdir);
	/* 将调用 grub-mkimage 来生成 core.img，其实最终调用它的主要函数 grub_install_generate_image */
	grub_install_make_image_wrap (/* source dir  */ grub_install_source_directory,
				/*prefix */ prefix,
				/* output */ imgfile,
				/* memdisk */ NULL,
				have_load_cfg ? load_cfg : NULL,
				/* image target */ mkimage_target, 0);

	/* core.img 已经生成到 boot directory 中，完成了大约一半的工作。下面该把 boot.img 也放到里面，
	 * 过程比较简单，仅是从 image directory 中 copy 到 boot directory:
	 * /usr/lib/grub/i386-pc/boot.img --> /boot/grub/i386-pc/boot.img */

	{
	  char *boot_img_src = grub_util_path_concat (2,
		  				  grub_install_source_directory,
						  "boot.img");
	  char *boot_img = grub_util_path_concat (2, platdir,
					      "boot.img");
	  grub_install_copy_file (boot_img_src, boot_img, 1);
	  /*  Now perform the installation. */
	  /* install_bootsector 默认是1，所以默认将调用 grub-bios-setup 的主函数来安装 boot.img 和 core.img */
	  if (install_bootsector)
	    grub_util_bios_setup (platdir, "boot.img", "core.img",
				  install_drive, force,
				  fs_probe, allow_floppy, add_rs_codes);
	}

这就是 grub-install 的过程，大部分的代码都是为了最后调用 grub-mkimage 和 grub-bios-setup 做准备。


## How linux kernel is booted


## APPENDIX

### 常见汇编指令说明

#### CMP

比较源操作数和目的操作数，并根据结果设置 EFLAGS 寄存器的 status flags。所谓比较是指： 目的操作数 - 源操作数(结果被丢弃)。Jcc 指令常常跟在 CMP 后，根据刚刚 CMP 的结果进行条件跳转。举例(AT&T汇编语法，第一个操作数是源操作数)：

	cmp a, b
	jg lable

意味着：jump to label if and only if b > a

#### TEST

将两个操作数做逻辑云