# Early Memory setup on x86

setup_arch 函数中有大量关于 memory 的函数，是时候复习一下，捋清思路。Kernel 启动过程中使用的 memory management 机制叫 memblock, 更早之前使用的是 bootmem 的机制。

做 memory management 的 prerequisites 是什么？ 可想而知是掌握了系统 memory 的 layout 信息，哪儿些可用，哪儿些不可用，这是 setup_arch 中 memory setup 的核心工作内容：获得 RAM layout. RAM layout source 有几个渠道: BIOS E820 带来一部分，放在 boot_params 中； Linux/X86 boot protocol 中的 setup data 可能有一部分; 还有一些 Layout 数据是前二者不知道的，如 kernel 自身占据的空间，要 kernel 自己处理。

RAM layout 信息的定义有两种角度： E820 & memblock, 使用 memblock 管理 memory, 二者如何 correlated? 带分析完代码再来总结。

  1. 找到完整的，健全的 E820 信息。E820 信息是 BIOS(or EFI) 对于 RAM 空间 layout 的认知。
  2. 将 E820 中的信息交给 memblock 管理
  3. E820 不知道的 RAM layout 信息也要交给 memblock 管理，包括：kernel 自身占据空间，boot protocol 中部分数据，如 setup data,
  4. BIOS E820 不知道的 RAM layout, 在 kernel 中知道后，也要补充到 kernel 的 E820 数据中。

对于 Linux kernel 而言，早期 memory setup 的内容主要是获取 RAM 所在的物理地址空间信息，交给 memblock 管理。本文的内容**应该**都来自 setup_arch 中的 callee function.

setup_arch 中第一个函数是 memblock_reserve, 直接就进入了 memblock 的领域，没有什么初始化代码？节奏好像有点快？原来 memblock 的初始化是静态的：
```
/* 为方便阅读，做了显示效果的优化 */

#define INIT_MEMBLOCK_REGIONS			128
#define INIT_PHYSMEM_REGIONS			4

#ifndef INIT_MEMBLOCK_RESERVED_REGIONS
	#define INIT_MEMBLOCK_RESERVED_REGIONS	INIT_MEMBLOCK_REGIONS
#endif

static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS];
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_RESERVED_REGIONS];
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS];
#endif

struct memblock __initdata_memblock memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,	/* empty dummy entry */
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.memory.name		= "memory",

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,	/* empty dummy entry */
	.reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,
	.reserved.name		= "reserved",

#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	.physmem.regions	= memblock_physmem_init_regions,
	.physmem.cnt		= 1,	/* empty dummy entry */
	.physmem.max		= INIT_PHYSMEM_REGIONS,
	.physmem.name		= "physmem",
#endif

	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```
看过 memblock 各个数据结构以及它的 introduction(memblock.c 文件的头部), 会发现它的设计很简单。

memblock 将 RAM 占据的物理地址空间分为 2 个 type：

  - memory: kernel 可用的空间
  - reserved: 已被占用的空间

Tip: RAM 的物理地址空间是不连续的。

一段物理地址空间用 struct memblock_region 表示：
```
/**
 * struct memblock_region - represents a memory region
 * @base: physical address of the region
 * @size: size of the region
 * @flags: memory region attributes
 * @nid: NUMA node id
 */
struct memblock_region {
	phys_addr_t base;
	phys_addr_t size;
	enum memblock_flags flags;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
	int nid;
#endif
};
```
每个 type 的空间由一系列静态分配好的 struct memblock_region 表示。 Early memory setup 的主要工作是找到所有这两种类型的空间，听起来很简单，但实际很繁琐。这些都是本文将要详细分析的。目前的观察：reserved 都是一个个 region 手动添加；"memory" 统一从 E820 那里添加，To be assure!

将摘取 setup_arch 中 memory setup 相关的部分分析，直接 inline 在代码中。

```
void __init setup_arch(char **cmdline_p)
{
	/*
	 * Reserve the memory occupied by the kernel between _text and
	 * __end_of_kernel_reserve symbols. Any kernel sections after the
	 * __end_of_kernel_reserve symbol must be explicitly reserved with a
	 * separate memblock_reserve() or they will be discarded.
	 */
	/* 暂时没必要理解 memblock_add_range 的细节，知道它会 merge overlapped regions
	 * into one 就够了。Kernel 自身占据的空间自然 goes to "reserved" type. 这里的
	 * 细节是，kernel itself 只 reserve 到 .brk section 之前, 见上面原 comments. */
	memblock_reserve(__pa_symbol(_text),
			 (unsigned long)__end_of_kernel_reserve - (unsigned long)_text);

	/*
	 * Make sure page 0 is always reserved because on systems with
	 * L1TF its contents can be leaked to user processes.
	 */
	/* 不懂 L1TF，但不妨事。*/
	memblock_reserve(0, PAGE_SIZE);

	/* 从 boot_params 中取得 initrd 占据的物理地址空间， add to "reserved" type. 阿*/
	early_reserve_initrd();
	...

	/* Kernel 使用的 E820 信息被组织为 3 个数据结构: e820_table, e820_table_firmware,
	 * e820_table_kexec, 这三个变量的介绍在 e820.c 的文件头部。e820_table 是 main table,
	 * 所有渠道来的 E820 信息经过 sanity check 后放入此 table.
	 * Kernel 的 struct e820_table 可容纳更多(E820_MAX_ENTRIES) entry, 因为 NUMA
	 * node 可能带来更多 entry, 而  中是 BIOS 来的 E820 信息，
	 * 只有 128 个, 远远不够。
	 *
	 * 此处将 boot_params.e820_table 转存到 main table: e820_table, 并由函数
	 * e820__update_table 对其 sanitize: sort; remove overlap (type 一样则直接
	 * merge; 不一样则需决定 overlap range 的 type: 取大数值的 type，保守的做法).
	 * 然后 copy 到另两个 table 中。 */
	e820__memory_setup();

	/* Linux/X86 boot protocol 中有一个 setup_data field, 是 pointer, 指向类型为
	 * struct setup_data 的 single linked list. setup data 可包含多种类型的数据,
	 * refer: SETUP_NONE.
	 * 扩展思考：谁会来构造 setup data of boot protocol? 目前的认知：boot loader 负责
	 * 构造 boot_params, 理应也由它构造 setup data, 但 grub 中未看到此操作，所以，有理由
	 * 认为，只有 EFI 会构造此数据。  待找到确定证据！？
	 *
	 * 本文中我们只关心 SETUP_E820_EXT, 其数据格式自然也是 struct boot_e820_entry.
	 * 处理的过程和 e820__memory_setup 一模一样，只是多了一步 early ioremap/inunmap
	 * 的操作，因为要将 setup data 中的物理地址 page map 为虚拟地址，才能在代码中使用。 */
	parse_setup_data();

	...

#ifdef CONFIG_MEMORY_HOTPLUG
	/*
	 * Memory used by the kernel cannot be hot-removed because Linux
	 * cannot migrate the kernel pages. When memory hotplug is
	 * enabled, we should prevent memblock from allocating memory
	 * for the kernel.
	 *
	 * ACPI SRAT records all hotpluggable memory ranges. But before
	 * SRAT is parsed, we don't know about it.
	 *
	 * The kernel image is loaded into memory at very early time. We
	 * cannot prevent this anyway. So on NUMA system, we set any
	 * node the kernel resides in as un-hotpluggable.
	 *
	 * Since on modern servers, one node could have double-digit
	 * gigabytes memory, we can assume the memory around the kernel
	 * image is also un-hotpluggable. So before SRAT is parsed, just
	 * allocate memory near the kernel image to try the best to keep
	 * the kernel away from hotpluggable memory.
	 */
	/* 待分析！*/
	if (movable_node_is_enabled())
		memblock_set_bottom_up(true);
#endif
	...

	/* after early param, so could get panic from serial */
	/* 刚刚解析过的 setup data of boot protocol 也要 add to “reserved” memblock. */
	memblock_x86_reserve_range_setup_data();
	...

	/* 上文已说，setup data of boot protocol 是 BIOS E820 不知道的 RAM layout，所以
	 * 现在将其 update 到 kernel 的 E820 数据中，目的是告诉 kernel: setup data 所在
	 * RAM 空间是 kernel 不可用的。
	 * Update 细节：只有 setup data 区域在 BIOS E820 中 type = E820_TYPE_RAM 时，
	 * 才将该区域 type 改为 : E820_TYPE_RESERVED_KERN. Update 后也要 sanitize.
	 * NOTE: 只 update e820_table & e820_table_kexec, 没有 e820_table_firmware,
	 * 需理解这三个 table 的目的，才知道为什么这样做。*/
	e820__reserve_setup_data();

	/* E820 原始数据的获取看起来已 over, 若 kernel parameter 有 mem= 或 memmap=,
	 * 限制 kernel 可用的 RAM, 那么 kernel 的 E820 数据也要相应更新，更新操作在 kernel
	 * parameter 的 callback 函数中完成，但那里只有 e820__range_remove/add, sanitize
	 * 的操作(e820__update_table)在这里完成。*/
	e820__finish_early_params();
	...

	/* 确保 kernel 所处区域在 BIOS E820 中须是 E820_TYPE_RAM, 若不是，则在 kernel 的
	 * E820 数据中做修正。 本质上也是对 E820 数据的 sanitize.
	 * 想来也有道理，对 BIOS 而言，它不知道 kernel 会被加载哪儿里，但那块区域肯定不是 BIOS
	 * 要用的，所以从 BIOS 角度，这块区域是自由的，available to OS. */
	e820_add_kernel_range();

	/* Kernel 的 E820 数据还需修正一下：根据实践得知的 2 个 special case. 详见函数内
	 * comments.
	 * comments 中的一些 words 需要 clarify: 1st 4KB of physical address 是 RAM,
	 * 但其数据属于 BIOS, 且一直有用，所以它应是 E820_TYPE_RESERVED; 640K - 1M of
	 * physical address 映射到 ROM 的 BIOS(system BIOS 或 BIOS extensions) 或
	 * video RAM, 也不是 kernel 可用的 RAM.  Great reference for memory map:
	 *   https://wiki.osdev.org/Memory_Map
	 *
	 * 修正好 E820 数据依然需要对它 sanitize.
	 */
	trim_bios_range();
	...

	/* partially used pages are not usable - thus we are rounding upwards */
	/* X86 架构中，memory 被划分为 4Kb 的 page frame 进行管理，pfn = page frame number.
	 * Note: Generally speaking, memory 并不单指 RAM. 但本文中提到 pfn 时，都是指
	 * RAM 这种 memory 的 pfn.
	 *
	 * 扩展知识：CPU 的 max pfn 自然和最大的物理地址有关，即 physical-address width of
	 * processor. Quick Knowledge:
	 *   - X86_32: 32-bit max
	 *   - X86_32 + PAE: 36-bit max
	 *   - X86_64: 46-bit max
	 *   - X86_64 + 5-level paging: 52-bit max
	 *
	 * 在 kernel 的 E820 数据中找到 E820_TYPE_RAM 类型的最大 pfn, 也即:
	 *     max_pfn = kernel 可用 RAM 空间的最大 pfn.
	 * 表示作为可用 RAM 物理空间的上边界。
	 */
	max_pfn = e820__end_of_ram_pfn();

	/* update e820 for memory not covered by WB MTRRs */
	/* MTRR 基础知识见下文。
	 *
	 * MTRR 由 firmware 初始化，kernel 读取来初始化自己内部的 MTRR 数据。过程繁琐，省略分析。
	 * 结果：MTRR init 后，kernel 有了 sanitized MTRR 数据。
	 *
	 * 由上面原 comments 可知：kernel 认为所有 RAM ranges 必须是 Write Back 类型，若
	 * E820 中某 range 不是 WB, 则需从 E820 数据中 trim it. So, essentially,
	 * MTRR initialization is for inspecting the kernel's E820 data. */
	mtrr_bp_init();

	/* MTRR trim 的过程依然很繁琐。由于 MTRR 不是本文的重点，暂且知道上述其目的即可。
	 * Trim 后，再重新找一次 RAM's max_pfn. NOTE: 此函数 dedicated to Intel MTRR. */
	if (mtrr_trim_uncached_memory(max_pfn))
		max_pfn = e820__end_of_ram_pfn();

	/* 这个赋值似乎透露出点什么？ */
	max_possible_pfn = max_pfn;

#ifdef CONFIG_X86_32
	...
#else
	/* need this before calling reserve_initrd */
	/* 若刚得到的 max_pfn 大于物理地址 4G，则用 max_low_pfn 来记录 4G 以下的最大 pfn. */
	if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
		max_low_pfn = e820__end_of_low_ram_pfn();
	else
		max_low_pfn = max_pfn;

	/* max_pfn 对应的 direct mapping 的虚拟地址记录在 high_memory 中。*/
	high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
#endif

	/* 下面是一堆 add range to "reserved"  memblock 的操作。 */

	/* Find and reserve possible boot-time SMP configuration */
	/* mptable 若存在，RESERVED it. */
	find_smp_config();

	/* iSCSI 在 memory 中也有数据，RESERVED it. */
	reserve_ibft_region();
	/* Need to conclude brk, before e820__memblock_setup(). it could use
	 * memblock_find_in_range, could overlap with brk area. */
	/* .brk section 已被使用，RESERVED it.*/
	reserve_brk();
	...

	/* 第一次设置 memblock.current_limit. commit dd7dfad7fb29 将其设为 1M, 不理解。 */
	memblock_set_current_limit(ISA_END_ADDRESS);

	/* 文章开头已说明：所有 RAM 空间都要交给 memblock 进行管理。memblock 将所有 RAM 空间
	 * 分为可用的 "memory" 和不可用的 "reserved" 两类，本函数 memory management 的
	 * 重点工作是找到这两类 RAM 的 range, 交给 memblock.
	 * 至今，E820 数据已经得到充分的 sanitization, 可以将其中 kernel 可用的 range 交给
	 * memblock 管理。NOTE: region 的 addr & size 都要 PAGE_SIZE 对齐。
	 *
	 * Q: Why treat E820_TYPE_RESERVED_KERN as E820_TYPE_RAM in 72d7c3b33c980,
	 * and added to memblock.memory? RESERVED_KERN is already in memblock.reserved.
	 * 推测：E820_TYPE_RESERVED_KERN 是 boot protocol 的 setup data, data 使用
	 * 完了，它占据的空间就可以用作它用，典型的例子： EXT E820 信息拿来初始化 kernel 内部
	 * 的 E820 数据后，就没用了。
	 *
	 * 一点新发现：region 可同时位于 "memory" & "reserved", refer __next_mem_range
	 * 的 comments. 在 "memory" 中 allocate, 分配的 range 同时记录在 "reserved",
	 * 防止 allocated area 被再次分配。
	 */
	e820__memblock_setup();

	/* 继续 RESERVE 不可用 RAM region. */
	reserve_bios_regions();
	...

	/* preallocate 4k for mptable mpc */
	/* 第一次 memory allocation 的机会，depends on configuration. */
	e820__memblock_alloc_reserved_mpc_new();
	...

	/* arch/x86/realmode 下的 image 将被放在物理地址 1M 以内，allocate & reserve it. */
	reserve_real_mode();

	/* 特定平台的 range reserve, 不必关心。*/
	trim_platform_memory_ranges();

	/* 上文函数 trim_bios_range 已将 E820 数据更新，且有 comments 预言这里的处理：在
	 * memblock.reserved 中 reserve 1M 内的部分 RAM 空间。*/
	trim_low_memory_range();

	/* Direct page mapping RAM. 非常复杂的内部处理。函数中有赋值 max_pfn_mapped.
	 * 猜测：应该会 skip non-RAM range 的 memory hole, 因为这里只 direct mapping
	 * RAM 的物理地址空间。*/
	init_mem_mapping();

}




```

### MTRR

Memory Type Range Registers(MTRR) 的权威介绍在 Intel SDM 3a, chapter 11.11, Page Attribute Table(PAT) 机制配合 MTRR 一起管理 memory type, 所以也需了解 PAT, 在 Intel SDM 3a, chapter 11.12. 代码中会看到，只有 MTRR enabled，才会初始化 PAT(mtrr_bp_pat_init).

MTRR quick start:

>A mechanism for associating memory types with physical address ranges in system memory, allow processor optimize operations for different memory such as RAM, ROM, frame-buffer memory, memory-mapped I/O devices. Up to 96 memory ranges can be defined, 88 among them are fixed ranges which are for the 1st Megabytes, the rest are for variable ranges. Following a hardware reset, all the fixed and variable MTRRs are disabled, make all physical memory uncacheable. Typically, firmware, BIOS for example, will configure the MTRRs. MTRR supports 5 memory types via 8-bit: Uncacheable(UC) 00H, Write Combining(WC) 01H, Write-through(WT) 04H, Write-protected(WP) 05H, Writeback(WB) 06H. Region's base address & size in MTRR must be 4K aligned.  MTRR feature is identified via CPUID, in case the CPU support it, additional info will be available in 64-bit read-only IA32_MTRRCAP MSR.

> MTRR is configured via three groups of registers: IA32_MTRR_DEF_TYPE MSR, fixed-range MTRRs, and variable range MTRRs.  IA32_MTRR_DEF_TYPE sets the default properties of the regions of physical memory that are not encompassed by MTRRs.

A [simple reference](https://www.kernel.org/doc/html/latest/x86/mtrr.html) for MTRR usage in Linux kernel

PAT quick start:
> PAT is a companion feature to the MTRRs, MTRR maps memory type to regions of physical address space, while PAT maps memory types to pages within linear address space. PAT extends the function of PCD & PWT bit of page table to allow 6(5 of MTRR, a new Uncached(UC-)) types to be assigned dynamically to pages of linear address. With PAT feature, PAT bit of page-table or page-directory entry works with PCD, PWT bit, to define the memory type for the page. NOTE: PAT bit exist only when entry maps to page frame. PAT involves 2 kinds of encoding: Memory type encoding used to program IA32_PAT MSR[1]; PAT+PCD+PWT encoding as the index to PAT entry of IA32_PAT MSR[2];

  1. Intel SDM 3a: Table 11-10. Memory Types That Can Be Encoded With PAT.
  2. Intel SDM 3a: Table 11-11. Selection of PAT Entries with PAT, PCD, and PWT Flags.

In case of any type conflicts(inside MTRR or between MTRR & PAT, etc): CPU choose the conservative memory type.

Programming MTRR 和 IA32_PAT MSR 是需要一些考虑的。MSR 属于 CPU，SMP 中，每个 CPU 都有 MTRR & PAT MSR，因此需要保证其值在所有 CPU 上的一致性。这也许就是 mtrr_bp_pat_init 中看起来不相关的复杂操作的原因。参考：11.11.8 MTRR Considerations in MP Systems, 11.12.4 Programming the PAT.

Variable Range MTRR 中 region range 的计算目前还没明白，暂且记下其 mask 计算原则：
> Address_Within_Range AND PhysMask = PhysBase AND PhysMask
