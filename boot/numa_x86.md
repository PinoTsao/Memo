# x86 NUMA

Refer to introduction of Non-uniform memory access(NUMA) at:

  1. [NUMA on wikipedia](https://en.wikipedia.org/wiki/Non-uniform_memory_access)
  2. Documentation/vm/numa.rst
  3. man 7 numa

According to the acronym itself & the manual, NUMA emphasizes on memory, that is why *Documentation/vm/numa.rst* says:

>Linux on X86 will "hide" any node representing a physical cell that has no memory attached, and reassign any CPUs attached to that cell to a node representing a cell that does have memory.

In other words, to Linux kernel, node with memory is genuine.

>NOTE:

>CPU & Memory hotplug 是通过 hot-pluggable Node 完成，并非(笔者原以为的)直插 CPU or memory. [这里](https://lwn.net/Articles/84089/)的定义中， Node 和 NUMA Node 并不完全一致，但自从 NUMA node is hotpluggable 后，**个人推测**业界 CPU & Memory hotplug 的实现都是通过 NUMA Node. Subtle difference between two definition: NUMA system 不止一条 system bus.(CPU 到 Memory 的连接是一条 system bus.)

NUMA capable motherboard in realy world:

![NUMA Node](numanode.png)

网摘: [NUMA "全书"](https://software.intel.com/en-us/articles/a-brief-survey-of-numa-non-uniform-memory-architecture-literature)

## NUMA Initialization

NUMA is information of hardware topology. Only firmware(or vendor) knows about the details: How many CPU &  memory a specific node. On modern PC, these info is passed OS via ACPI's SRAT & SLIT.

In case hardware & Linux kernel support NUMA, its initialization on X86 is thru: setup_arch --> initmem_init. For NUMA+X86_64, initmem_init 的实现是 arch/x86/mm/numa_64.c 中的这位，我们只分析它:

```
void __init initmem_init(void)
{
	x86_numa_init();
}

/* Try each configured NUMA initialization method until one succeeds.  The
 * last fallback is dummy single node config encompassing whole memory and
 * never fails. */
/* According to comments above & a glance at function body, 可看出 NUMA 的初始化
 * 分 3 种不同方式: common X86, AMD-specific, NO NUMA(emulate 1 node).
 *
 * What is AMD-specific?
 * AMD 在其第一款 64 bit CPU, Opteron of 2003, 最先支持 NUMA. According to head
 * comments of amdtopology.c, 其 NUMA 信息(memory range : NodeID)存储在
 * northbridge 的 PCI config space, 读取它构建 Kernel 的 NUMA data. I.e., AMD
 * NUMA 的初始设计不通过 ACPI table 传递 NUMA 信息。随着 Intel 开始支持 NUMA in late
 * 2007, 大家开始一致使用 ACPI table 的方式传递 NUMA 信息。
 *
 * In case both platform & kernel support NUMA but failed to initialize NUMA due to
 * corrupted SRAT or SLIT data, it make sense that Linux kernel regard that all CPU
 * & memory belongs to dummy node.
 */
void __init x86_numa_init(void)
{
	if (!numa_off) {
#ifdef CONFIG_ACPI_NUMA
		/* OS 只有通过 ACPI(SRAT & SLIT) 才知道硬件上的 NUMA 拓扑 */
		if (!numa_init(x86_acpi_numa_init))
			return;
#endif

#ifdef CONFIG_AMD_NUMA
		/* Old style AMD Opteron NUMA detection, SKIP */
		if (!numa_init(amd_numa_init))
			return;
#endif
	}

	/* In case 硬件架构不支持 NUMA, kernel 配置不支持 NUMA，或上面的 NUMA 初始化失败，
	 * we come to "dummy_numa_init": 把已知的所有 memory range(0 - max_pfn) 作为
	 * 一个 memory block. */
	numa_init(dummy_numa_init);
}

static int __init numa_init(int (*init_func)(void))
{
	int i;
	int ret;

	/* MAX_LOCAL_APIC denotes the max APIC ID, NOT the max CPU number kernel support.
	 * Refer to:
	 *     http://lkml.iu.edu/hypermail/linux/kernel/1509.3/01126.html
	 *
	 * Tips:
	 *   1. 一个 APIC ID 对应一个 CPU, 一个 CPU 对应一个 NODE. CPU 的 APIC ID
	 *      (register/ACPI 中各有一份)可能 non-contiguous;
	 *   2. MAX_NUMNODES <-- NODES_SHIFT <-- CONFIG_NODES_SHIFT 的定义关系?
	 *      autoconf.h 不是为 .c 直接 #include 用, 而 MAX_NUMNODES 方便直接使用。
	 *
	 * Node 管理与 CPU 一样使用 bitmap --> mask 的形式。
	 * 此初始化似乎多余？因为声明处已初始化。
	 */
	for (i = 0; i < MAX_LOCAL_APIC; i++)
		set_apicid_to_node(i, NUMA_NO_NODE);

	/* node_states[NR_NODE_STATES] 定义处，元素 N_POSSIBLE 的初始化(NODE_MASK_ALL)
	 * 有些 tricky: 因 MAX_NUMNODES 可能不是 BITS_PER_LONG 的整数倍, 所以使用
	 * BITMAP_LAST_WORD_MASK set bitmap 中剩余 bit.
	 * enum node_states 的定义值得分析, 看起来和 memory zone 相关，TBD.
	 *
	 * Clear all nodemask variables.
	 */
	nodes_clear(numa_nodes_parsed);
	nodes_clear(node_possible_map);
	nodes_clear(node_online_map);

	/* According to NR_NODE_MEMBLKS, it consider a NUMA node has 2 memory ranges
	 * at most; But according to E820_MAX_ENTRIES，it consider a NUMA node has 3
	 * ranges at most.  WHY?
	 */
	memset(&numa_meminfo, 0, sizeof(numa_meminfo));

	WARN_ON(memblock_set_node(0, ULLONG_MAX, &memblock.memory,
				  MAX_NUMNODES));
	WARN_ON(memblock_set_node(0, ULLONG_MAX, &memblock.reserved,
				  MAX_NUMNODES));

	/* In case that parsing SRAT failed. */
	/* memblock.memory is not aware of region hot-pluggable attribute. */
	WARN_ON(memblock_clear_hotplug(0, ULLONG_MAX));

	/* numa_distance[] 中的数据来自 SLIT's matrix entry, 表示 node 之间距离。
	 * numa_distance_cnt 自然表示 node 数量。但似乎没必要 reset？初始化前还有人 set 它？ */
	numa_reset_distance();

	/* Common X86 NUMA 初始化: ACPI's SRAT & SLIT 解析。见下文 x86_acpi_numa_init */
	ret = init_func();
	if (ret < 0)
		return ret;

	/*
	 * We reset memblock back to the top-down direction
	 * here because if we configured ACPI_NUMA, we have
	 * parsed SRAT in init_func(). It is ok to have the
	 * reset here even if we did't configure ACPI_NUMA
	 * or acpi numa init fails and fallbacks to dummy
	 * numa init.
	 */
	memblock_set_bottom_up(false);

	/* 此刻 numa_meminfo 中的数据还是 SRAT 来的 raw data， sanitize it. 详见下文。*/
	ret = numa_cleanup_meminfo(&numa_meminfo);
	if (ret < 0)
		return ret;

	/* 模拟 NUMA node, 默认关闭。待有需要再分析。 Skip. */
	numa_emulation(&numa_meminfo, numa_distance_cnt);

	/* 做了几件事情：
	 * 1. 根据 sanitized NUMA memory info, set nid into memblock.memory's region;
	 * 2. memblock.reserved 中 region 所在 node 上的所有 region 都要 clear
	 *    MEMBLOCK_HOTPLUG flag;
	 * 3. 检查 NUMA memory range 是否 cover 了 memblock.memory range;
	 * 4. 为每个合格 node 分配 pg_data_t 空间;
	 * 详见下文。*/
	ret = numa_register_memblks(&numa_meminfo);
	if (ret < 0)
		return ret;

	/* 此时 nr_cpu_ids 的值还是来自 config 或 kernel parameter(因为是 early param,
	 * 所以会生效，if present), 尚未被修改为系统实际支持的 CPU 数；且 x86_cpu_to_node_map
	 * 也未初始化(它的初始化在后面的 init_cpu_to_node)， so, why check here?
	 * defensive coding/checking???
	 * 假如有 NodeID, 检查 NodeID 是否属于刚 set online 的 ID 之一? if NO, clear
	 * NodeID for this CPU.  注意：nr_cpu_ids 是 kernel 定义的逻辑 CPU ID. */
	for (i = 0; i < nr_cpu_ids; i++) {
		int nid = early_cpu_to_node(i);

		if (nid == NUMA_NO_NODE)
			continue;
		if (!node_online(nid))
			numa_clear_node(i);
	}

	/* 看看此函数 comments, 我就说 nr_cpu_ids 还没有真正初始化为实际值。详见下文。*/
	numa_init_array();

	return 0;
}

/* 解析 ACPI 中 NUMA 相关的 table: SRAT & SLIT. */
int __init x86_acpi_numa_init(void)
{
	int ret;

	/* Parse SRAT & SLIT. Details as following. */
	ret = acpi_numa_init();
	if (ret < 0)
		return ret;
	return srat_disabled() ? -EINVAL : 0;
}

/* drivers/acpi/numa/srat.c */
int __init acpi_numa_init(void)
{
	int cnt = 0;

	if (acpi_disabled)
		return -EINVAL;

	/* Should not limit number with cpu num that is from NR_CPUS or nr_cpus=
	 * SRAT cpu entries could have different order with that in MADT.
	 * So go over all cpu entries in SRAT to get apicid to node mapping. */

	/* Kernel 配置的 CPU 数目(config 或 parameter) 不应限制 SRAT 的解析，因为
	 * 二者完全不相关，前者表示支持的数量，后者表中是 APIC ID. */

	/* SRAT: System Resource Affinity Table */
	/* SRAT handler acpi_parse_srat does nothing but get acpi_srat_revision.
	 * Leave all the work to following subtable handler.
	 * SRAT 的 subtable 中只定义了 4 种 structure: 3 CPU & 1 memory*/
	if (!acpi_table_parse(ACPI_SIG_SRAT, acpi_parse_srat)) {
		struct acpi_subtable_proc srat_proc[3];

		/* 此初始化函数位于 drivers/acpi, 所有 arch 通用，所以 sub-table handler
		 * 覆盖所有 possible CPU, 当前 CPU 只可能是其中一种。
		 * GICC(Generic Interrupt Controller CPU) 属于 ARM.
		 *
		 * Sub-table 的主要内容是 APIC ID & 它对应的 proximity domain ID. PXM ID
		 * 是 ACPI 术语(32-bit), 对应 kernel 使用的逻辑 NUMA node ID(MAX_NUMNODES),
		 * sub-table handler 将二者映射，也将 APIC ID 与 逻辑 NUMA node ID 映射。
		 * Kernel 限制 max PXM ID 为 MAX_NUMNODES, 若 ACPI's PXM ID > MAX_NUMNODES,
		 * 则该 PXM 不会被 mapped with a node ID, i.e., kernel 不使用它。
		 * 注意 "数目(count)" 与"ID" 的区别。
		 *
		 * ACPI's PXM ID 与 kernel's 逻辑 NUMA ID 通过: pxm_to_node_map[] &
		 * node_to_pxm_map[] 互相 mapping: 对每一个 PXM ID, nodes_found_map 中
		 * first unset bit's index 即是对应的逻辑 NUMA ID. PXM ID 与 逻辑 NUMA ID
		 * 一一对应。 APIC ID 与 kernel 逻辑 NUMA ID 通过 __apicid_to_node[] 映射.
		 *
		 * nodes_found_map 与 numa_nodes_parsed 看起来功能一样，区别？前者是
		 * driver/acpi 内部使用，后者是 x86 内部使用。
		 * NOTE: 一个 PXM/NUMA node 上可有多个逻辑 CPU, 即 APIC ID.
		 *
		 * In general: kernel 用逻辑 NUMA ID 来 identify ACPI 中的 PXM ID.
		 */
		memset(srat_proc, 0, sizeof(srat_proc));
		srat_proc[0].id = ACPI_SRAT_TYPE_CPU_AFFINITY;
		srat_proc[0].handler = acpi_parse_processor_affinity;
		srat_proc[1].id = ACPI_SRAT_TYPE_X2APIC_CPU_AFFINITY;
		srat_proc[1].handler = acpi_parse_x2apic_affinity;
		srat_proc[2].id = ACPI_SRAT_TYPE_GICC_AFFINITY;
		srat_proc[2].handler = acpi_parse_gicc_affinity;

		acpi_table_parse_entries_array(ACPI_SIG_SRAT,
					sizeof(struct acpi_table_srat),
					srat_proc, ARRAY_SIZE(srat_proc), 0);

		/* Memory affinity sub-table provides following topology info:
		 *   1. physical address range and its PXM;
		 *   2. whether or not the memory range is hot-pluggable.
		 *
		 * 同样，SRAT 中的 memory range 也要 map 到 kernel's 逻辑 NUMA ID.
		 * Get kernel's 逻辑 NUMA ID from ACPI‘s PXM ID via the same routine
		 * as above, then store range info into numa_meminfo. Parsing 结束也会
		 * set numa_nodes_parsed 一下。 Besides, 若 memory range is hot-pluggable,
		 * mark it in memblock region's flag. parsed_numa_memblks 记录 memory
		 * affinity sub-table 数。
		 * NOTE: 会再次 evaluate max_possible_pfn according to each memory range,
		 * 看来可能 NUMA memory 信息与 E820 不一致。之前的 evaluation 基于 E820.
		 */
		cnt = acpi_table_parse_srat(ACPI_SRAT_TYPE_MEMORY_AFFINITY,
					    acpi_parse_memory_affinity, 0);
	}

	/* After processing SRAT, kernel is aware about the following relationship:
	 *   1. kernel's 逻辑 NUMA ID <--> ACPI's PXM ID
	 *   2. APIC ID --> kernel's 逻辑 NUMA ID
	 *   3. memory range --> kernel's 逻辑 NUMA ID
	 * 所以， MAX_LOCAL_APIC & MAX_NUMNODES 都是对 ID 的限制，而不是 number, ID 可能
	 * 不连续的！但 ACPI driver 分配的 "逻辑 NUMA ID" 是连续的。
	 */

	/* SLIT: System Locality Information Table */
	/* SLIT 以正方形矩阵的形式提供了 PXM 间的*相对距离*信息。 Relative distance 意味着
	 * memory latency, SLIT 中用一个 byte 表示它。0xFF 表示 2 个 node/PXM domain 间
	 * 不可达; SLIT 定义 node/PXM domain 自己到自己的距离是 10. 所以 Relative distance
	 * 的有效范围是 [10 - 255). NOTE: 为什么用矩阵形式? 因为 node[i->j] 的距离可能 ！=
	 * node [j->i] 的距离。另：SLIT 中有 PXM domain 的数量信息，所以理论上 PXM ID 从 0
	 * 开始且连续.
	 * slit_valid() 根据 ACPI spec 对 matrix 做简单验证：
	 *   1. PXM 自己到自己的 distance 必须是 LOCAL_DISTANCE;
	 *   2. PXM 之间的距离必须 > LOCAL_DISTANCE;
	 *
	 * SLIT parsing: 将 "最大逻辑 NUMA ID" 记在 numa_distance_cnt(其实是 ID), 根据
	 * 它分配 memory for numa_distance(并简单初始化为 LOCAL_DISTANCE & REMOTE_DISTANCE),
	 * 读取 SLIT matrix entry, set into numa_distance[] correspondingly.
	 *
	 * 有必要再次分析 numa_alloc_distance, 因为产生了和 474aeffd88b8 一样的想法，但它
	 * 又被 b678c91aefa7 revert.
	 * 2019/12/24 update: 再次阅读发现上述想法似乎并不准确。核心前提：NUMA node 必有
	 * memory, 未必有 CPU. 如文初所述， Linux kernel 会隐藏无 memory 的 node.
	 * 先解析 CPU sub-table, 确定 kernel's 逻辑 NUMA ID 和 ACPI's PXM ID 的映射关系；
	 * 再解析 Memory sub-table, 在上述映射关系中找到逻辑 NUMA ID, set numa_meminfo.
	 * 二者的解析都会 set numa_nodes_parsed(这看来是问题所在, 混在一起，Mess), 而
	 * numa_meminfo 中记录的逻辑 NUMA ID 是包含 CPU & memory 的 sane PXM domain.
	 * 代码的做法: 取 numa_nodes_parsed 的 *copy*, 根据 numa_meminfo 中的 nid
	 * set *copy*, 最终根据 copy 中 NUMA ID 来分配 numa_distance[] 使用的 memory.
	 * (没有只有 memory 的 node 吗?)
	 */
	acpi_table_parse(ACPI_SIG_SLIT, acpi_parse_slit);

	if (cnt < 0)
		return cnt;
	else if (!parsed_numa_memblks)
		return -ENOENT;
	return 0;
}

/* max_pfn 从 E820 获得，作为 range limit, WHY? 知识点补漏：kernel 的 e820_table *应*
 * cover 所有 NUMA memory, 这点从该函数逻辑可看出。另外， BIOS 有理由/能力知道 NUMA node's
 * E820 信息，并将该信息传给 kernel. 所以该函数的实际作用算是交叉验证 NUMA memory 的有效性：
 * 根据 kernel's E820 验证 SRAT's Memory Affinity Structure 中数据的有效性。因为
 * kernel's E820 数据已 sanitized.
 * 目测此函数是为了各种 buggy system 而存在，sane system 不应该存在 range 不一致的情况。*/
int __init numa_cleanup_meminfo(struct numa_meminfo *mi)
{
	const u64 low = 0;
	const u64 high = PFN_PHYS(max_pfn);
	int i, j, k;

	/* first, trim all entries */
	for (i = 0; i < mi->nr_blks; i++) {
		struct numa_memblk *bi = &mi->blk[i];

		/* make sure all blocks are inside the limits */
		/* start 的判断看起来多余。u64 不会小于 0. */
		bi->start = max(bi->start, low);
		bi->end = min(bi->end, high);

		/* and there's no empty or non-exist block */
		/* e820__memblock_setup() 已将 e820_table 加入 memblock.memory 管理，所以
		 * numa_meminfo 中的 range 必与 memblock.memory 重叠，否则 range is buggy.
		 */
		if (bi->start >= bi->end ||
		    !memblock_overlaps_region(&memblock.memory,
			bi->start, bi->end - bi->start))
			numa_remove_memblk_from(i--, mi);
	}

	/* merge neighboring / overlapping entries */
	/* numa_meminfo 中 entry order 和 SRAT 一致。这里判断的组合：nid x range overlap
	 * 所以有 4 condition: 1. nid 相同 + overlap; 2. nid 相同 + !overlap;
	 *                    3. nid 不同 + overlap; 4. nid 不同 + !overlap;
	 */
	for (i = 0; i < mi->nr_blks; i++) {
		struct numa_memblk *bi = &mi->blk[i];

		for (j = i + 1; j < mi->nr_blks; j++) {
			struct numa_memblk *bj = &mi->blk[j];
			u64 start, end;

			/* See whether there are overlapping blocks.  Whine about but
			 * allow overlaps of the same nid.  They will be merged below. */
			if (bi->end > bj->start && bi->start < bj->end) {
				/* (相邻 range 重叠 & nid 不同) = NUMA 初始化失败. 看来 Linux 不
				 * 接受："不同 node 之间有 memory range 重叠" 这样 insane 的情况。
				 * 排除 3rd condition.
				 */
				if (bi->nid != bj->nid) {
					/* SKIP pr_err */
					return -EINVAL;
				}
				/* 否则如原注释所说，merge 同 nid 的 overlapped range. Warning 先.*/
				/* SKIP pr_warn */
			}

			/* 3 condition left: 相邻 range 不重叠(2,4) or 重叠 with 同 nid(1). */

			/* Join together blocks on the same node, holes between
			 * which don't overlap with memory on other nodes. */
			/* nid 不同意味 range 不重叠(2,4)， i.e., 这组相邻 range 在不同 node 上
			 * 且 memory range 不重叠，是合理情况，所以 continue 下一组相邻 range. */
			if (bi->nid != bj->nid)
				continue;

			/* 只剩 condtion: overlap ranges with same nid(1). Merge range 先 */
			start = min(bi->start, bj->start);
			end = max(bi->end, bj->end);
			/* 再 loop: 对 merged range, 是否还有 "nid 不同" & "overlapped" range?
			 * 如上所说，这是 Linux kernel 不能接受的情况，将视为 NUMA 初始化失败。 */
			for (k = 0; k < mi->nr_blks; k++) {
				struct numa_memblk *bk = &mi->blk[k];

				if (bi->nid == bk->nid)
					continue;

				/* nid 不同， range 重叠，则跳出 for(). */
				if (start < bk->end && end > bk->start)
					break;
			}
			/* 若还有(第三个)不同 nid 的 range 与 merged range overlap, 则 continue
			 * 回到开头，*将在 return -EINVAL 处返回*。 但本函数*代码逻辑成立的前提*：SRAT
			 * 中的 entry(I.e., numa_meminfo 中的 entry) 以 NodeID 为序，即，同 node
			 * 上的 range entry 在 SRAT 中摆在一起。  REALLY? */
			if (k < mi->nr_blks)
				continue;

			/* 现在只剩 condition: 相邻 block 的 range 有 overlap, 将真正 merge. */
			/* SKIP printk, too long to fit typography */

			/* Merge: 删掉后一个 block. */
			bi->start = start;
			bi->end = end;
			numa_remove_memblk_from(j--, mi);
		}
	}

	/* clear unused ones */
	for (i = mi->nr_blks; i < ARRAY_SIZE(mi->blk); i++) {
		mi->blk[i].start = mi->blk[i].end = 0;
		mi->blk[i].nid = NUMA_NO_NODE;
	}

	return 0;
}

static int __init numa_register_memblks(struct numa_meminfo *mi)
{
	unsigned long uninitialized_var(pfn_align);
	int i, nid;

	/* Account for nodes with cpus and no memory */
	/* 唯一初始化 node_possible_map 处。如上面原注释所说，numa_nodes_parsed 包含了
	 * no memory 的 node. */
	node_possible_map = numa_nodes_parsed;

	/* 再次碰到此函数。参考： 474aeffd88b8 & https://lkml.org/lkml/2017/4/6/632
	 * 不同的是，此刻的 numa_meminfo 已 sanitized.
	 * 矛盾的是，此处用 ARRAY_SIZE(mi->blk) 遍历， 而下面 for() 用 mi->nr_blks. */
	numa_nodemask_from_meminfo(&node_possible_map, mi);
	if (WARN_ON(nodes_empty(node_possible_map)))
		return -EINVAL;

	/* 根据 sanitized NUMA memory info, set Node ID into memblock.memory's region.
	 * NOTE: NUMA memory(RAM) range info 与 E820/memblock.memory (RAM too) range info
	 * 可能不完美一致。Refer to comments of numa_meminfo_cover_memory().
	 */
	for (i = 0; i < mi->nr_blks; i++) {
		struct numa_memblk *mb = &mi->blk[i];
		memblock_set_node(mb->start, mb->end - mb->start,
				  &memblock.memory, mb->nid);
	}

	/* At very early time, the kernel have to use some memory such as
	 * loading the kernel image. We cannot prevent this anyway. So any
	 * node the kernel resides in should be un-hotpluggable.
	 *
	 * And when we come here, alloc node data won't fail. */
	/* 首先，根据 ACPI NUMA memory info, set node ID into memblock.reserved's region;
	 * 然后，根据 reserved region's node ID 构造 reserved_nodemask(下一步使用);
	 * 最后，clear MEMBLOCK_HOTPLUG flag of region in node of reserved_nodemask,
	 * 因为之前 reserved 的 region 都是 kernel 自身使用的数据，所以不可热插拔。*/
	numa_clear_kernel_node_hotplug();

	/* If sections array is gonna be used for pfn -> nid mapping, check
	 * whether its granularity is fine enough.  不懂，TBD.	 */
#ifdef NODE_NOT_IN_PAGE_FLAGS
	pfn_align = node_map_pfn_alignment();
	if (pfn_align && pfn_align < PAGES_PER_SECTION) {
		printk(KERN_WARNING "Node alignment %LuMB < min %LuMB, rejecting NUMA config\n",
		       PFN_PHYS(pfn_align) >> 20,
		       PFN_PHYS(PAGES_PER_SECTION) >> 20);
		return -EINVAL;
	}
#endif

	/* 对 ACPI's NUMA memory range 数据从不同角度轮番 sanity check.
	 *
	 * 这次根据 E820/memlbock.memory 的 range 信息。  隐含的知识：Linux kernel 认为
	 * memblock.memory 中的 range 是 kernel 的所有真正可用 RAM range, 以此为基准
	 * verify 其他渠道(NUMA) 来的 memory(RAM) range 信息。(memblock.memory 又是从
	 * sanitized kernel's E820 来)*/
	if (!numa_meminfo_cover_memory(mi))
		return -EINVAL;

	/* Finally register nodes. */
	for_each_node_mask(nid, node_possible_map) {
		u64 start = PFN_PHYS(max_pfn);
		u64 end = 0;

		for (i = 0; i < mi->nr_blks; i++) {
			if (nid != mi->blk[i].nid)
				continue;
			start = min(mi->blk[i].start, start);
			end = max(mi->blk[i].end, end);
		}

		if (start >= end)
			continue;

		/* Don't confuse VM with a node that doesn't have the
		 * minimum amount of memory */
		if (end && (end - start) < NODE_MIN_SIZE)
			continue;

		/* 终极目标： 为合格 node 分配 pg_data_t, 放入 node_data[] 管理。 Obviously,
		 * 优先在 node 自身 memory 上分配该空间。(pg_data_t 是另一个宏大话题) 分配了该
		 * structure 的 node 也被 set online in node_online_map/node_states[N_ONLINE],
		 * BUT, online node 的逻辑 NodeID *可能是* 不连续的 */
		alloc_node_data(nid);
	}

	/* Dump memblock with node info and return. */
	memblock_dump_all();
	return 0;
}

/* Sanity check to catch more bad NUMA configurations (they are amazingly
 * common).  Make sure the nodes cover all memory. */
/* 看来 NUMA memory range info 和 E820 不一致的情况还很多 */
static bool __init numa_meminfo_cover_memory(const struct numa_meminfo *mi)
{
	u64 numaram, e820ram;
	int i;

	/* __absent_pages_in_range 的返回值 *nr_absent* 以 pfn 为单位表示：在 [mi->blk[i]]
	 * 上 & 不在 [memblock.memory 同 node] 上的 range(size).  该函数的 comments:
	 *     Return the number of holes in a range on a node.
	 * 表明只有 NUMA range 大于 kernel 实际可用 RAM range 的 delta 才叫 hole.
	 * 若 NUMA range 小于实际 RAM range 时，nr_absent 是负值，加到 numaram 上，反而
	 * correct 了该数据。   正常情况应是： NUMA range > memblock.memory range.
	 *
	 * 所以，numaram 表示: ACPI NUMA 视角的 RAM size(pfn 数). */
	numaram = 0;
	for (i = 0; i < mi->nr_blks; i++) {
		u64 s = mi->blk[i].start >> PAGE_SHIFT;
		u64 e = mi->blk[i].end >> PAGE_SHIFT;
		numaram += e - s;
		numaram -= __absent_pages_in_range(mi->blk[i].nid, s, e);
		if ((s64)numaram < 0)
			numaram = 0;
	}

	/* 背景： maxpfn 包含 memory hole, 因为 RAM 的物理地址不连续。
	 * e820ram 表示 E820 视角的 RAM size(pfn 数). */
	e820ram = max_pfn - absent_pages_in_range(0, max_pfn);

	/* We seem to lose 3 pages somewhere. Allow 1M of slack. */
	/* Linux kernel 从 firmware 拿到 E820 数据，经各种 sanitation 后，将这份 E820 数据
	 * 看作 system 上的可用 RAM, (不管实际 hardware 上插了多少 RAM.)然后交给 memblock
	 * 管理。这个 sanitation 过程必然是及其保守的，所以才可能 lose some pages(memory hole
	 * 变大). 理论上，没有 lose some pages 的话，这两个数据应该相等，实际 e820ram 可能更小，
	 * 但 numaram 不应该更小，若如此，说明 NUMA memory 数据有错误。
	 * 题外话： Kernel 从外部拿到的任何数据，都不会轻易相信，要做各种 sanitation, 这是
	 * 理智的行为。
	 *
	 * 1 << (20 - PAGE_SHIFT) = 1M/4k, 即 1M 的 4k page frame 数. 看来 Linux
	 * kernel 允许 NUMA memory(RAM) info 有 1M 的误差(slack), 误差大于 1M，说明
	 * NUMA 数据*非常*不可靠，初始化失败。
	 */
	if ((s64)(e820ram - numaram) >= (1 << (20 - PAGE_SHIFT))) {
		printk(KERN_ERR "NUMA: nodes only cover %LuMB of your %LuMB e820 RAM. Not used.\n",
		       (numaram << PAGE_SHIFT) >> 20,
		       (e820ram << PAGE_SHIFT) >> 20);
		return false;
	}
	return true;
}

/* There are unfortunately some poorly designed mainboards around that
 * only connect memory to a single CPU. This breaks the 1:1 cpu->node
 * mapping. To avoid this fill in the mapping for all possible CPUs,
 * as the number of CPUs is not known yet. We round robin the existing
 * nodes. */
/*
 * 不明白前一半 comments, 但 code 的确是践行后一半 comments. Round robin online node,
 * assign NodeID to each possible CPU in nr_cpu_ids. 和上面相邻代码有相同问题： 此前
 * 前并 set NodeID 给 CPU, so, 此处用意用处何在？
 * 注意： node_online_map 可以不连续 */
static void __init numa_init_array(void)
{
	int rr, i;

	rr = first_node(node_online_map);
	for (i = 0; i < nr_cpu_ids; i++) {
		if (early_cpu_to_node(i) != NUMA_NO_NODE)
			continue;
		numa_set_node(i, rr);
		rr = next_node_in(rr, node_online_map);
	}
}
```
NUMA 自身的初始化终于结束，最后关于 nr_cpu_ids 的部分尚未详细分析，因为还不知道 CPU 的初始化是怎样(下一节的内容)；另外， NUMA node 与 CPU 之间的映射关系也需要在函数 init_cpu_to_node 中确定，也涉及 CPU 的初始化。所以，至此，先跳转去下一节分析 CPU 初始化，再来分析 init_cpu_to_node.

终于回来了～ 回忆下 NUMA 和 CPU 初始化，核心内容之一是各种 ID 的映射。从 ACPI 角度来看，它使用 APIC ID 识别 CPU, 使用 PXM ID 识别 NUMA node, 但 kernel 要使用自己的 logical CPU ID 和 Node ID, 所以有了各种 mapping 工作。这些 mapping 工作是理解函数 *init_cpu_to_node* 的必要条件。

Background: kernel 最终使用自己的 [Logical CPU ID:Node ID] 的 mapping, 这要做一些转换工作。解析 ACPI MADT 时(acpi_process_madt -> acpi_process_madt -> generic_processor_info)， 确定了 [Logical CPU ID:APIC ID] mapping in *x86_cpu_to_apicid*; 解析 SRAT 时(x86_numa_init -> x86_numa_init -> acpi_numa_init -> acpi_numa_processor_affinity_init)，确定了 [APIC ID:Node ID] mapping in *__apicid_to_node*, 所以顺理成章有了 [logical CPU ID:Node ID] mapping in *x86_cpu_to_node_map*, 即：

>[Logical CPU ID:APIC ID] + [APIC ID:Node ID] --> [Logical CPU ID:Node ID]

```
/*
 * Setup early cpu_to_node.
 *
 * Populate cpu_to_node[] only if x86_cpu_to_apicid[],
 * and apicid_to_node[] tables have valid entries for a CPU.
 * This means we skip cpu_to_node[] initialisation for NUMA
 * emulation and faking node case (when running a kernel compiled
 * for NUMA on a non NUMA box), which is OK as cpu_to_node[]
 * is already initialized in a round robin manner at numa_init_array,
 * prior to this call, and this initialization is good enough
 * for the fake NUMA cases.
 *
 * Called before the per_cpu areas are setup.
 */
/* 函数名中的 cpu, node, 指自己定义的 logical CPU ID, 和 logical Node ID. */
void __init init_cpu_to_node(void)
{
	int cpu;
	u16 *cpu_to_apicid = early_per_cpu_ptr(x86_cpu_to_apicid);

	/* 确认 APIC ID 有 map 到 logical CPU ID. */
	BUG_ON(cpu_to_apicid == NULL);

	for_each_possible_cpu(cpu) {
		/* 根据 logical CPU ID 拿到对应的 Node ID, 正是要 mapping 的内容。*/
		int node = numa_cpu_node(cpu);

		if (node == NUMA_NO_NODE)
			continue;

		if (!node_online(node))
			init_memory_less_node(node);

		/* map logical CPU ID to Node ID*/
		numa_set_node(cpu, node);
	}
}
```
## NUMA 相关之 CPU 初始化

Here "CPU 初始化" is about "CPU ID" or "CPU number". The concept of "CPU ID" is frequently seen in NUMA initialization, it is one of prerequisite to fully understand NUMA code. The same as NUMA, it is firmware who tell the info of CPU ID to OS via ACPI MADT.

"CPU ID" has 2 levels of definition:

  1. hardware perspective: it is **APIC ID**;
  2. software perspective: **logical CPU ID** defined by kernel itself.

Linux kernel will map **APIC ID** to **logical CPU ID**.
 本节的内容从 ACPI 的初始化开始。

A question: ACPI table 是 firmware 动态生成, or 静态存在(工程师将其与 firmware build 在一起烧写进 ROM)? 最新结论： firmware 动态生成 ACPI table, but w/ prerequisites： firmware is provided device-specific descriptor, 然后 firmware 中的 ACPI agent 根据这些信息构造 ACPI table. 代码自身无法检测硬件平台的信息。  References:

  1. https://patents.google.com/patent/US7502803B2/en: Abstract
  2. Intel SDM 3a, 8.4.3 MP Initialization Protocol Algorithm for MP Systems: 5. As part of the boot-strap code, the **BSP creates an ACPI table** and/or an MP table and adds its initial APIC ID to these tables as appropriate

setup_arch() 中，CPU & NUMA 相关的初始化函数的出现顺序是：

  - acpi_boot_table_init
  - early_acpi_boot_init
  - initmem_init: NUMA 初始化
  - ...
  - acpi_boot_init
  - get_smp_config
  - init_apic_mappings()
  - prefill_possible_map()
  - init_cpu_to_node()

可见 ACPI 的初始化散落在 3 个函数(按出现顺序):

  1. acpi_boot_table_init
  2. early_acpi_boot_init
  3. acpi_boot_init

三个函数的名字初看有些困惑，不明白为何要分成 3 个函数进行初始化，他们分别要做什么。待分析完，这个问题便知。首先解惑一点: 这 3 处 ACPI 初始化都处于 kernel booting 阶段,所以函数名中都带有 "acpi", "boot", "init" 字眼。

```c
/*
 * acpi_boot_table_init() and acpi_boot_init()
 * called from setup_arch(), always.
 *	1. checksums all tables
 *	2. enumerates lapics
 *	3. enumerates io-apics
 *
 * acpi_table_init() is separate to allow reading SRAT without
 * other side effects.
 *
 * side effects of acpi_boot_init:
 *   acpi_lapic = 1 if LAPIC found
 *   acpi_ioapic = 1 if IOAPIC found
 *   if (acpi_lapic && acpi_ioapic) smp_found_config = 1;
 *   if acpi_blacklisted() acpi_disabled = 1;
 *   acpi_irq_model=...
 *   ...
 */
/* 上面的 comments is unfriendly, 且看起来已 outdated.
 *
 * 函数的主要工作内容顾名思义，根据 memory 中的 ACPI table 初始化 kernel 内部数据结构:
 * struct acpi_table_desc initial_tables[ACPI_MAX_TABLES], 后续所有 table parsing
 * 都从此 structure 中寻找 table. Besides, some miscellaneous work is done here:
 * Simple Boot Flag Table 解析； 多 MADT 检查，有没有，有的话使用哪儿个 MADT. */
acpi_boot_table_init()
{
	/* 特定型号的 PC 不支持 acpi, 或 acpi 中的部分功能，他们被 blacklisted 在
	 * acpi_dmi_table, 通过 DMI 信息识别这些 PC.*/
	dmi_check_system(acpi_dmi_table);

	/* If acpi_disabled, bail out */
	/* 若 acpi 被 disable, 则不必 run 数量庞大的 ACPI code, (省事儿了) 这是 bail out
	 * 的含义。什么情况下会 acpi_disabled?
	 *   1. 特定型号 PC: acpi_dmi_table 中第一项
	 *   2. acpi=off, kernel parameter
	 */
	if (acpi_disabled)
		return;

	/* Initialize the ACPI boot-time table parser. */
	if (acpi_table_init()) {
		disable_acpi();
		return;
	}

	/* Simple Boot Flag Table: 别处定义的 Industry spec, simply reserved by ACPI.
	 * 本文下笔时最新版本是 2.1.
	 * Quick knowledge:
	 * On PC-AT BIOS computer, a BOOT register is defined in main CMOS memory.
	 * (typically accessed through I/O ports 70h/71h on Intel Architecture platforms)
	 * The location of the BOOT register in CMOS is indicated by a BOOT table
	 * found via the ACPI RSDT table.
	 * 是一个 OS 通知 firmware 的机制, 在 CMOS 中的 1-byte register 写数据, firmware
	 * 运行时检查其中的 bit. */
	acpi_table_parse(ACPI_SIG_BOOT, acpi_parse_sbf);

	/* blacklist may disable ACPI entirely */
	/* 又是特定型号 and/or 特定版本 BIOS 的 ACPI 不能正常工作. Besides, 还有其他 ACPI
	 * 兼容性的 check, 忽略。*/
	if (acpi_blacklisted()) {
		if (acpi_force) {
			printk(KERN_WARNING PREFIX "acpi=force override\n");
		} else {
			printk(KERN_WARNING PREFIX "Disabling ACPI support\n");
			disable_acpi();
			return;
		}
	}
}

/* acglobal.h */
ACPI_GLOBAL(struct acpi_table_list, acpi_gbl_root_table_list);

/* 函数位于 driver/acpi/table.c */
int __init acpi_table_init(void)
{
	acpi_status status;

	/* kernel parameter: acpi_force_table_verification */
	if (acpi_verify_table_checksum) {
		pr_info("Early table checksum verification enabled\n");
		acpi_gbl_enable_table_validation = TRUE;
	} else {
		pr_info("Early table checksum verification disabled\n");
		acpi_gbl_enable_table_validation = FALSE;
	}

	/* kernel 内部用 struct acpi_table_desc 来描述 ACPI table. Linux kernel 使用
	 * 全局变量(静态分配) initial_tables[] 管理 root table array(其他 OS 未必，这是
	 * acpi_initialize_tables 第三个入参 "allow_resize" 的含义所在, 因为 ACPI 是跨
	 * 平台的). ACPI 内部用 acpi_gbl_root_table_list 管理 root table array.
	 *
	 * 首先寻找 RSDP. For x86, it is via callback: x86_default_get_root_pointer,
	 * 以前此 callback 是空函数，自从 team member 的 patch 合入后，就可通过它快捷拿到
	 * RSDP; 否则
	 *   - 若是 EFI firmware, 则可快捷的拿到，
	 *   - 若是 BIOS firmware, 则老老实实的 scan memory.
	 * Scan RSDP in memory 很简单，按 spec 说明即可，对 BIOS 来说，RSDP 可能出现在:
	 *   - First 1KB of EBDA, or
	 *   - BIOS read-only memory space [0xE0000h, 0xFFFFF).
	 *   (Reference: ACPI SPEC, 5.2.5.1 Finding the RSDP on IA-PC Systems)
	 *
	 * NOTE: 地址 [0xE0000h, 0xFFFFF) 中不仅可能有 ACPI table, 还可能有 SMBIOS.
	 *
	 * 若 ACPI 通过 scan memory(acpi_find_root_pointer) 寻找 RSDP, 则要使用 fixmap
	 * 中的 __early_ioremap, 将"待搜寻物理地址" map 为虚拟地址才好访问，访问结束 unmap.
	 * 实际代码过程较复杂，暂无需深究。
	 *
	 * 找到 RSDP 后按部就班解析各 tables(acpi_tb_parse_root_table), 因为 table 中的
	 * 地址数据都是物理地址，仍要 map 为虚拟地址才能读取 memory. 过程依然繁琐，也无需深究。
	 * 调用此函数后，kernel 内部变量 initial_tables[] 被初始化完成。*/
	status = acpi_initialize_tables(initial_tables, ACPI_MAX_TABLES, 0);
	if (ACPI_FAILURE(status))
		return -EINVAL;

	/* 背景知识： Documentation/admin-guide/acpi/initrd_table_override.rst
	 * initrd 带来的 ACPI tables 也要初始化到 ACPI 内部管理数据结构。*/
	acpi_table_initrd_scan();

	/* 吐槽：不得不说，acpi 的代码写的太太繁琐了！想找到变量赋值的地方要绕很久！*/

	/* ACPI driver 中有 validate/invalidate table 的概念，其实是 ACPI 管理 table
	 * reference 的机制。 Simply speaking: (使用 table 虚拟地址)access table 时，
	 * table reference count++; 使用结束则 count--. Count 为 0 时表示无人使用它，将
	 * unmap 其虚拟地址。 Refer: acpi_get_table/acpi_put_table
	 * (WHY 释放? 不需要将 ACPI table 永久映射访问？)
	 *
	 * table 的虚拟地址记录在 acpi_table_desc.pointer, IIRC, 此虚拟地址在 fixmap 区域。
	 */

	/* Background: 有些 buggy ACPI 可能提供 2 个 MADT. Check 是否有多个 MADT, 若有，
	 * 则要确认使用哪儿一个。
	 * acpi_apic_instance 是 kernel parameter, 表示 kernel 要使用第几个 MADT,
	 * default to 0.
	 *   - acpi_apic_instance = 2 表示 use 2nd MADT, if available;
	 *   - acpi_apic_instance = 1 or 0 表示 use 1st MADT. */
	check_multiple_madt();
	return 0;
}
```

从软件视角看， APIC is a bunch of registers, 其中只有一个是 MSR: IA32_APIC_BASE, 其他都是 memory mapped registers, 即映射在 CPU 物理地址空间，这些 memory mapped registers 有一个 base address, 由 Intel SDM 3a, 10.4.1 知: it defaults to 32-bit APIC_DEFAULT_PHYS_BASE.

>**NOTE**： x2APIC archetecture 既可以工作在  x2APIC mode, 也可以工作在 APIC(xAPIC) mode. x2APIC mode 时，其 APIC registers 都是 MSR.

所以可推断：若是 x2APIC arch, 其 MADT header 中依然有 APIC base address, 在其工作在 APIC mode 时用。

第二个 ACPI 初始化函数 early_acpi_boot_init 的重点内容是解析 MADT header, 找到 memory mapped APIC 寄存器的 base address.

```
int __init early_acpi_boot_init(void)
{
	/* If acpi_disabled, bail out */
	if (acpi_disabled)
		return 1;

	/* Process the Multiple APIC Description Table (MADT), if present */
	/* 找到 APIC register 的 base address, 若是 !x2apic 则映射到 fixmap area 中预留
	 * 的虚拟地址空间; 拿到 boot CPU's APIC ID & APIC version. 详见下文。*/
	early_acpi_process_madt();

	/* Hardware-reduced ACPI mode initialization: */
	/* FADT 中有 1 个 bit indicate if it is HW reduced ACPI.(ACPI_FADT_HW_REDUCED)
	 * 上面 acpi_table_init 初始化内部 table array 时，已对 FADT 解析并初始化了
	 * acpi_gbl_reduced_hardware.
	 *
	 * 对 Hardware-reduced APCI 的理解参考 ACPI spec, 3.11.1 Hardware-reduced ACPI.
	 * 看起来是说，对于 PC 架构, ACPI 定义了一些了标准的硬件，但某些 platform 并未实现为
	 * 标准 PC.
	 * 从代码(acpi_generic_reduced_hw_init)推测, 未按标准 PC arch 实现的 platform
	 * 包括： 没有 HPET, 不支持 PIC, ISA interrupt.
	 */
	acpi_reduced_hw_init();

	return 0;
}

/* 变量名顾名思义：从 ACPI(MADT) 来的 local APIC register base address. */
#ifdef CONFIG_X86_LOCAL_APIC
	static u64 acpi_lapic_addr __initdata = APIC_DEFAULT_PHYS_BASE;
#endif

/* Early process MADT 的主要工作是找到 memory mapped APIC registers 的基地址。
 * 方式： MADT header 中的 32-bit 物理地址； 或 MADT sub-table “Local APIC Address
 * Override” 中的 64-bit 物理地址，如果有的话。
 *
 * Besides, APIC driver 的架构值的一看。(暂不知根据什么分类) APIC 的物理实现有多种， 每种
 * 对应一个 APIC driver, 由 macro: apic_driver 声明。放在 .apicdrivers section.
 * 参考： apic.h 中 apic_driver 的 comments.
 * 所以 booting 阶段需要 probe 当前系统是哪儿一种 APIC. 看起来 acpi_parse_madt -->
 * default_acpi_madt_oem_check 中第一次 probe 当前系统的 APIC. 根据经验，普通 PC 的
 * APIC 应该是 apic_flat_64.c 中的 apic_flat, 下面以他为例分析。
 */
static void __init early_acpi_process_madt(void)
{
#ifdef CONFIG_X86_LOCAL_APIC
	int error;

	/* 将 MADT 中获得的 APIC registers 基地址记录在 acpi_lapic_addr 中。代码繁琐，
	 * 只有函数 register_lapic_address 值得分析。 TBD: 何时操作 IA32_APIC_BASE? */
	if (!acpi_table_parse(ACPI_SIG_MADT, acpi_parse_madt)) {

		/* Parse MADT LAPIC entries */
		/* 返回值的处理 confusing, 有三种情况: < 0 表示错误; 0 表示没有相应 entry;
		 * > 0 表示处理的 entry 数。*/
		error = early_acpi_parse_madt_lapic_addr_ovr();
		if (!error) {
			/* 函数执行成功，说明拿到 APIC register base address, ID, version 等信息，
			 * so, ACPI MADT 中有 local APIC info. */
			acpi_lapic = 1;
			/* 若系统存在 MP table, 则 find_smp_config -> xx -> smp_scan_config 时
			 * 已 set. 正常的 SMP 系统，MP table, ACPI table 只存在一个*/
			smp_found_config = 1;
		}
		if (error == -EINVAL) {
			/* Dell Precision Workstation 410, 610 come here. */
			printk(KERN_ERR PREFIX
			       "Invalid BIOS MADT, disabling ACPI\n");
			disable_acpi();
		}
	}
#endif
}

/* 下面函数的分析，引出(我的)一个问题：目前观察到 3 处 APIC ID: PowerOn/Reset 时 MP
 * INITIALIZATION, 每个 CPU core 根据拓扑得到 unique initial APIC ID; cpuid 指令可
 * 得到 APIC ID； ACPI's MADT 中有 APIC ID, 三者是什么关系？
 * 已确认: MP INITIALIZATION 时得到的叫做 *initial APIC ID*, 生成后写入 APIC ID 寄存器，
 * 某些 CPU 允许更改此寄存器的内容； cpuid 指令得到的是始终是不变的 *initial APIC ID*;
 *
 * 而 ACPI MADT 中的 APIC ID 呢？目前推测，BIOS 生成 ACPI table 时从寄存器拿到 ID 写入。
 *
 * Refer:
 *   1. Intel SDM 3a, 8.4.5 Identifying Logical Processors in an MP System
 *   2. Intel SDM 3a, 10.4.6 Local APIC ID
 */

/* 入参是从 ACPI 中的 APIC registers base address.
 * 找到 APIC register base address 后需要做什么呢？因为是物理地址, 对于 memory mapped
 * registers, 则需要映射到虚拟地址空间才能 access. 但其中还有细节，见 inline comments. */
void __init register_lapic_address(unsigned long address)
{
	mp_lapic_addr = address;

	/* x2apic_mode 在 setup_arch --> check_x2apic 中已初始化。
	 *
	 * 根据 x2apic mode 是否 enable , 有 2 个情况：
	 *   - !x2apic mode： APIC registers are accessed via mapped to physical
	 *     address; then, registers are accessed in code via virtual address
	 *     of page-mapping.
	 *     fixmap 区域已为 APIC 预留 virtual address range: FIX_APIC_BASE.
	 *
	 *   - x2apic mode: APIC registers 都是 MSR. 通过指令 wrmsr/rdmsr 访问，不需
	 *     memory map.
	 *
	 *  Refer: Intel SDM 3a, 10.12.1.2 x2APIC Register Address Space
	 */
	if (!x2apic_mode) {
		set_fixmap_nocache(FIX_APIC_BASE, address);
		apic_printk(APIC_VERBOSE, "mapped APIC to %16lx (%16lx)\n",
			    APIC_BASE, address);
	}

	/* 第一个执行这段代码的 CPU 是 boot CPU. 从 APIC register 读取 APIC ID/version.
	 * (大多数, 包括)apic_flat APIC driver 的 .read 都是 native_apic_mem_read().
	 *
	 * Refer: Intel SDM 3a, Figure 10-6. Local APIC ID Register;
	 *                      Figure 10-7. Local APIC Version Register
	 */
	if (boot_cpu_physical_apicid == -1U) {
		boot_cpu_physical_apicid  = read_apic_id();
		boot_cpu_apic_version = GET_APIC_VERSION(apic_read(APIC_LVR));
	}
}
```
前两个 ACPI 初始化函数看起来做的事情并不多。

按各函数在 setup_arch 中的出现顺序，之后是 NUMA 初始化， booting 阶段 ACPI 初始化的重点工作可想而知将落到第三个函数: acpi_boot_init. Let's get back to NUMA now.

～～～～～～～～～～～～～～～～～～～～～～

回来继续 ACPI 最后一个 early init 函数：

```
int __init acpi_boot_init(void)
{
	/* those are executed after early-quirks are executed */
	/* 依然是针对少数 buggy ACPI 的 PC. 略。*/
	dmi_check_system(acpi_dmi_table_late);

	/* If acpi_disabled, bail out	 */
	if (acpi_disabled)
		return 1;

	/* BOOT 在 acpi_boot_table_init() 中已处理，重复. 这行代码自 Linux kernel 导入 git
	 * 就存在, 少有 BOOT table 的环境，不便测试。Maintainer seems very cautious:
	 * https://lore.kernel.org/linux-pm/24266640.LfmLNjZWAc@kreacher/
	 */
	acpi_table_parse(ACPI_SIG_BOOT, acpi_parse_sbf);

	/* set sci_int and PM timer address. */
	/* FADT 在 acpi_boot_table_init - x - x - -> acpi_tb_parse_fadt 已解析， ACPI
	 * driver 将其 copy & format-convert 到变量 acpi_gbl_FADT, acpi_parse_fadt
	 * 中只是使用它。
	 *
	 * NOTE: FADT 针对 ARM 和 IA-PC 分别定义了 2-byte long flag(ARM_BOOT_ARCH,
	 * IAPC_BOOT_ARCH); PM(Power Management) timer 是 ACPI 标准定义的硬件，位于
	 * I/O address space, if present. */
	acpi_table_parse(ACPI_SIG_FADT, acpi_parse_fadt);

	/* Process the Multiple APIC Description Table (MADT), if present */
	/* 本函数的重点，解析 local APIC, I/O APIC sub-table, 详见下文。I/O APIC 涉及众多
	 * 中断知识细节，out of my current knowledge, 暂只能简要分析。*/
	acpi_process_madt();

	/* HPET Quick Knowledge: HPET spec 独立于 ACPI spec, 所有这类 spec 在
	 * https://uefi.org/acpi 中可找到。历史原因 HPET 在不同的 spec 中可能有不同的名字，
	 * 如：IA-PC HPET, Event timers, MultiMedia timer(MMT or MM timer).
	 * What is a timer? refer to the combination of a Counter, Comparator, and
	 * Match Register. The Comparator compares the contents of the Match
	 * Register against the value of a free running up-counter. When the output
	 * of the up-counter equals the value in the match register an interrupt
	 * is generated. IA-PC HPET Architecture allows up to 32 compare/match
	 * registers per counter. Each of the 32 comparators can output an interrupt.
	 *
	 * 简而言之，Spec 定义最多有 8 个 timer block, one timer block 最多产生 32 个
	 * timer interrupt(或者说有 32 个 timer), 当我们说 "HPET" 时，通常指一个 timer
	 * block. Timer block memory map 到物理地址空间，CPU 可直接寻址， 一个 timer block
	 * (HPET)的 registers 占据 1k 空间， base address is 4k aligned, registers are
	 * 64-bit/8-byte aligned.
	 * ACPI table 中只有一个 HPET table, 描述一个 HPET; 若系统中有多个 HPET, 其他 HPET
	 * 的信息在 ACPI namespace 中描述，SPEC 给的理由是：Only one HPET table is needed
	 * in order to boot strap the OS.
	 * HPET table 描述信息: Hardware Rev ID, timer 数量 in 1st timer block's,
	 * base address, register block ID/HPET number/sequence number(0 - 7?).
	 * 看起来有点奇怪，为什么是 "1st" HPET? HPET table 是 1st HPET 吗？
	 * 1 个 HPET 提供的 timer(32 max) 应足够 kernel 使用，以 Intel ICH9 为例，它提供
	 * 4 个 timer. HPET 位于南桥 chipset 上，同样以 Intel ICH9 为例，它的 base address
	 * 配置为 0xFED00000. 至此，我想可以 make an assumption: 目前所有 PC 都只有 1 个
	 * HPET, 位于地址 0xFED00000, 提供 3 或多个 timer, HPET table 用来描述它的信息。 */
	acpi_table_parse(ACPI_SIG_HPET, acpi_parse_hpet);
	/* Boot Graphics Resource Table, UEFI 限定。*/
	if (IS_ENABLED(CONFIG_ACPI_BGRT))
		acpi_table_parse(ACPI_SIG_BGRT, acpi_parse_bgrt);

	if (!acpi_noirq)
		x86_init.pci.init = pci_acpi_init;

	/* Do not enable ACPI SPCR console by default */
	/* Serial Port Console Redirection Table */
	acpi_parse_spcr(earlycon_acpi_spcr_enable, false);
	return 0;
}

/* early_acpi_process_madt() 从 MADT header 中读取 APIC register base address,
 * 这里 parse 每一个 sub-table.
 * 已发 cleanup patch: https://patchwork.kernel.org/patch/11346677/, 等待回应 */
static void __init acpi_process_madt(void)
{
#ifdef CONFIG_X86_LOCAL_APIC
	int error;

	/* acpi_parse_madt 在 early process MADT 时已调用过，重复！其中干货仅是根据 MADT
	 * header 初始化 acpi_lapic_addr. Early process MADT 时还会额外解析 Local APIC
	 * Address Override Structure.
	 * 若有 OVERRIDE structure, 这里 acpi_lapic_addr 岂不是可能再次 override?
	 * 答案：该变量只暂存 ACPI 读来的地址，最终通过 register_lapic_address() 记录在
	 * mp_lapic_addr.
	 */
	if (!acpi_table_parse(ACPI_SIG_MADT, acpi_parse_madt)) {
		/* Parse MADT 中所有 local APIC entries. 详见下文 */
		error = acpi_parse_madt_lapic_entries();
		if (!error) {
			/* early process 时已 set, 这里 set 应是无 override structure 的情况。*/
			acpi_lapic = 1;

			/* Local APIC parsing OK 才继续 parse I/O APIC. 详见下文 */
			mutex_lock(&acpi_ioapic_lock);
			error = acpi_parse_madt_ioapic_entries();
			mutex_unlock(&acpi_ioapic_lock);
			if (!error) {
				acpi_set_irq_model_ioapic();

				smp_found_config = 1;
			}
		}

		if (error == -EINVAL) {
			/*
			 * Dell Precision Workstation 410, 610 come here.
			 */
			printk(KERN_ERR PREFIX
			       "Invalid BIOS MADT, disabling ACPI\n");
			disable_acpi();
		}
	} else {
		/*
 		 * ACPI found no MADT, and so ACPI wants UP PIC mode.
 		 * In the event an MPS table was found, forget it.
 		 * Boot with "acpi=off" to use MPS on such a system.
 		 */
 		/* 若 ACPI 中没有 MADT，kernel 将认为该系统使用 UP PIC, 即使有 MP table, 也
 		 * forget it.*/
		if (smp_found_config) {
			printk(KERN_WARNING PREFIX
				"No APIC-table, disabling MPS\n");
			smp_found_config = 0;
		}
	}

	/*
	 * ACPI supports both logical (e.g. Hyper-Threading) and physical
	 * processors, where MPS only supports physical.
	 */
	/* 看起来存在一种 case: ACPI & MP table 都存在，但 ACPI MADT 提供 local APIC info,
	 * MP table 提供 I/O APIC info? */
	if (acpi_lapic && acpi_ioapic)
		printk(KERN_INFO "Using ACPI (MADT) for SMP configuration "
		       "information\n");
	else if (acpi_lapic)
		printk(KERN_INFO "Using ACPI for processor (LAPIC) "
		       "configuration information\n");
#endif
	return;
}

/* Parse MADT 中 local APIC sub-table.
 *
 * 背景: MADT 如何列出所有 local APIC structure? Spec 给出几条 guidelines:
 *   1. Boot processor is listed first
 *   2. For multi-threaded processors, BIOS should list the first logical
 *      processor of each of the individual multi-threaded processors in MADT
 *      before listing any of the second logical processors.
 *   3. APIC IDs < 0xFF should be listed in APIC subtable, APIC IDs >= 0xFF
 *      should be listed in X2APIC subtable.
 *
 * Refer: 5.2.12.1 MADT Processor Local APIC / SAPIC Structure Entry Order
 *      : 5.2.12.12 Processor Local x2APIC Structure, "Note:"
 * entry order example: https://lkml.org/lkml/2015/9/7/285
 *
 * 本函数的干货是拿到每一个 APIC ID, 注册/map 到 kernel 内部的 logical CPU ID. 当前代码
 * 逻辑是按照 APIC ID 在 MADT 中出现的顺序依次注册： acpi_register_lapic. (以前不是)
 */
static int __init acpi_parse_madt_lapic_entries(void)
{
	int count;
	int x2count = 0;
	int ret;
	struct acpi_subtable_proc madt_proc[2];

	if (!boot_cpu_has(X86_FEATURE_APIC))
		return -ENODEV;

	/* Q: Parse ACPI_MADT_TYPE_LOCAL_SAPIC first? 答案：
	 * SAPIC = Streamlined APIC: An advanced APIC commonly found on Intel
	 * Itanium TM Processor Family-based 64-bit systems. So, SAPIC 和 APIC/X2APIC
	 * 不会同时存在，因为他们是不同 CPU arch 上的 APIC. So, if NO SAPIC, check
	 * APIC/X2APIC, 逻辑没问题。
	 *
	 * SAPIC sub-table 中有多个 ID field: ID & EID(Extension ID) 一起描述 SAPIC's
	 * local APIC 好ID; 在 ACPI namespace 还有 ACPI processor ID 和 ACPI Processor
	 * UID, 前者在 ACPI Spec 中标记为 deprecated(APIC & X2APIC structure 中没有它),
	 * 但实际代码中, SAPIC 用它做 processor 的 ACPI ID.
	 *
	 * (APIC ID & APIC's ACPI ID) 经 acpi_register_lapic -> generic_processor_info
	 * 映射得到 kernel 视角的 logical CPU ID/number. 细节见下文。
	 */
	count = acpi_table_parse_madt(ACPI_MADT_TYPE_LOCAL_SAPIC,
				      acpi_parse_sapic, MAX_LOCAL_APIC);

	if (!count) {
		/* 确认不是 Itanium, then, process the other possibilities.
		 * NOTE: 支持 x2APIC arch 的 CPU 可以工作在 x2APIC mode 或 xAPIC mode 下，
		 * 注意这些术语(arch VS mode)间的 subtle difference. Q: 一个 CPU package
		 * 的不同 logical CPU 是否可分别处于两个不同 mode？直觉是 NO.  参考：
		 *   1. Intel SDM 3a, chapter 10.3.
		 *   2. x86_x64体系探索及编程(邓志), 18.1.2 APIC 体系的版本
		 *
		 * APIC VS xAPIC VS x2APIC: 最显著的不同可能是他们的 ID length，参考 Intel
		 * SDM 3a, Figure 10-6. Local APIC ID Register.
		 * 但 ACPI_MADT_TYPE_LOCAL_APIC 看起来可以 cover both APIC & xAPIC.
		 *
		 * APIC/X2APIC sub-table 很简单，干货只是 APIC ID & ACPI processor UID,
		 * sub-table handler 获得他们后，处理同 SAPIC: acpi_register_lapic -->
		 * generic_processor_info. 细节见下文。
		 * NOTE: CONFIG_X86_X2APIC 决定 Kernel 是否支持 x2APIC. */
		memset(madt_proc, 0, sizeof(madt_proc));
		madt_proc[0].id = ACPI_MADT_TYPE_LOCAL_APIC;
		madt_proc[0].handler = acpi_parse_lapic;
		madt_proc[1].id = ACPI_MADT_TYPE_LOCAL_X2APIC;
		madt_proc[1].handler = acpi_parse_x2apic;
		ret = acpi_table_parse_entries_array(ACPI_SIG_MADT,
				sizeof(struct acpi_table_madt),
				madt_proc, ARRAY_SIZE(madt_proc), MAX_LOCAL_APIC);
		if (ret < 0) {
			printk(KERN_ERR PREFIX
					"Error parsing LAPIC/X2APIC entries\n");
			return ret;
		}

		count = madt_proc[0].count;
		x2count = madt_proc[1].count;
	}

	if (!count && !x2count) {
		printk(KERN_ERR PREFIX "No LAPIC entries present\n");
		/* TBD: Cleanup to allow fallback to MPS */
		return -ENODEV;
	} else if (count < 0 || x2count < 0) {
		printk(KERN_ERR PREFIX "Error parsing LAPIC entry\n");
		/* TBD: Cleanup to allow fallback to MPS */
		return count;
	}

	/* Check if NMI connects to LINT1 of LAPIC or not, if not, just print
	 * warning. 简而言之： NMI 应该连接到 LINT1 pin.
	 *
	 * Take ACPI_MADT_TYPE_LOCAL_APIC_NMI for example, ACPI spec says:
	 * Each Local APIC NMI connection requires a separate Local APIC NMI
	 * structure. For example, if the platform has 4 processors with ID 0-3 and
	 * NMI is connected LINT1 for processor 3 and 2, two Local APIC NMI entries
	 * would be needed in the MADT. */
	x2count = acpi_table_parse_madt(ACPI_MADT_TYPE_LOCAL_X2APIC_NMI,
					acpi_parse_x2apic_nmi, 0);
	count = acpi_table_parse_madt(ACPI_MADT_TYPE_LOCAL_APIC_NMI,
				      acpi_parse_lapic_nmi, 0);
	if (count < 0 || x2count < 0) {
		printk(KERN_ERR PREFIX "Error parsing LAPIC NMI entry\n");
		/* TBD: Cleanup to allow fallback to MPS */
		return count;
	}
	return 0;
}

/* 省略 sub-table handler 函数的分析，但其中有一段 comments(w/ my massage), 值的留下：
 *
 *   We need to register disabled CPU as well to permit counting disabled CPUs.
 *   This allows us to size cpus_possible_map more accurately, and not to permit
 *   preallocating memory for all NR_CPUS when we use CPU hotplug.
 */

/* acpi_register_lapic - register a local APIC and generates a logic CPU number
 * @id: local APIC ID to register
 * @acpiid: ACPI ID to register
 * @enabled: this CPU is enabled or not
 *
 * Returns the logic cpu number which maps to the local APIC
 */
static int acpi_register_lapic(int id, u32 acpiid, u8 enabled)
{
	unsigned int ver = 0;
	int cpu;

	/* Tip again: MAX_LOCAL_APIC 是指 ID，不是数量。触发此 error patch 的实例：
	 * https://openbenchmarking.org/system/1210053-SU-SYSTER84874/syster1/dmesg */
	if (id >= MAX_LOCAL_APIC) {
		printk(KERN_INFO PREFIX "skipped apicid that is too big\n");
		return -EINVAL;
	}

	/* ACPI spec 6.1: this processor is unusable, and the operating system
	 * support will not attempt to use it. */
	if (!enabled) {
		++disabled_cpus;
		return -EINVAL;
	}

	/* 这两行似乎为了 fix warning on 32-bit: fb3bbd6a663fe. ToBeAnalyse. */
	if (boot_cpu_physical_apicid != -1U)
		ver = boot_cpu_apic_version;

    /* 核心函数： APIC ID mapping to logical CPU number. */
	cpu = generic_processor_info(id, ver);
	if (cpu >= 0)
		/* x86_cpu_to_acpiid 似乎没有实际用处？ TBD */
		early_per_cpu(x86_cpu_to_acpiid, cpu) = acpiid;

	return cpu;
}

int generic_processor_info(int apicid, int version)
{
	/* nr_cpu_ids 初始化自 CONFIG_NR_CPUS, or kernel parameter "nr_cpus".
	 * 表示 kernel 支持的最大 CPU 数。 */
	int cpu, max = nr_cpu_ids;

	/* boot_cpu_physical_apicid 已在 early_acpi_process_madt --> ... ->
	 * register_lapic_address 由 APIC ID register 初始化。本文条件下: X86_64 & SMP,
	 * it is NOT SET in phys_cpu_present_map.
	 *
	 * (start_kernel-->time_init 在 UniProcessor 情况下会 set.) */
	bool boot_cpu_detected = physid_isset(boot_cpu_physical_apicid,
				phys_cpu_present_map);

	/* Skip 无关紧要 codes. 背景：若系统存在 MP table, 那么 boot_cpu_physical_apicid
	 * 初始化后，到此函数之间，boot_cpu_physical_apicid 可能会被 MP table 中的值修改。*/

	/* If boot cpu has not been detected yet, then only allow up to
	 * nr_cpu_ids - 1 processors and keep one slot free for boot cpu.
	 *
	 * 进入此函数注册 apicid 时，若发现:
	 *   1. boot_cpu_physical_apicid is not set in phys_cpu_present_map, AND
	 *   2. 已注册的 num_processors > max config value, AND
	 *   3. 入参 apicid 也不是 boot_cpu_physical_apicid
	 * 则需在 phys_cpu_present_map 给 boot cpu 留个位置。disabled_cpus 表示 kernel
	 * 不能使用的 CPU 数目. "留个位置"就是不处理入参 apicid, 把它当作 disabled_cpu.
	 */
	if (!boot_cpu_detected && num_processors >= nr_cpu_ids - 1 &&
	    apicid != boot_cpu_physical_apicid) {
	    /* nr_cpu_ids 的值和 ACPI MADT 中 local APIC entry 数可能不同，若 ACPI MADT
	     * 中 entry 数 > kernel 支持的数目，则不支持多余 CPUs, 将他们计入 disabled_cpus.
	     * 所以 "thiscpu" is evaluated like this, 表示 CPU index.
	     * 本函数中多处定义 thiscpu, 看起来冗余，仅 debug 用。*/
		int thiscpu = max + disabled_cpus - 1;

		pr_warning(
			"APIC: NR_CPUS/possible_cpus limit of %i almost"
			" reached. Keeping one slot for boot cpu."
			"  Processor %d/0x%x ignored.\n", max, thiscpu, apicid);

		disabled_cpus++;
		return -ENODEV;
	}

	/* 逻辑同上，不赘述。*/
	if (num_processors >= nr_cpu_ids) {
		int thiscpu = max + disabled_cpus;

		pr_warning("APIC: NR_CPUS/possible_cpus limit of %i "
			   "reached. Processor %d/0x%x ignored.\n",
			   max, thiscpu, apicid);

		disabled_cpus++;
		return -EINVAL;
	}

	/* 逻辑很明显, BSP 要作为第一个 kernel logical CPU ID: 0, 其他的依次递增。
	 * NOTE: "logical cpuid" 是 kernel 定义的 CPU 编号(纯软件视角)，与 APIC ID 有
	 * 1:1 映射关系 via cpuid_to_apicid[].
	 */
	if (apicid == boot_cpu_physical_apicid) {
		/* x86_bios_cpu_apicid is required to have processors listed
		 * in same order as logical cpu numbers. Hence the first
		 * entry is BSP, and so on.
		 * boot_cpu_init() already hold bit 0 in cpu_present_mask for BSP. */
		/* 上面注释暂时不懂，TBD. */
		cpu = 0;

		/* Logical cpuid 0 is reserved for BSP. */
		cpuid_to_apicid[0] = apicid;
	} else {
		cpu = allocate_logical_cpuid(apicid);
		if (cpu < 0) {
			disabled_cpus++;
			return -EINVAL;
		}
	}

	/* SKIP Validate version code */

	if (apicid > max_physical_apicid)
		max_physical_apicid = apicid;

	/* 一个物理 CPU 在不同 context 下有不同的 ID 定义:
	 *   1. First, 它在硬件拓扑上有 APIC ID;
	 *   2. Then, 它在 ACPI namespace 中有 ACPI processor ID;
	 *   3. At last, 它在 Linux kernel 中有 logical CPU ID.
	 * ID 之间互相映射，比如这里，还有 caller: acpi_register_lapic 处理返回值的操作。
	 * PERCPU, TBD.
	 */
#if defined(CONFIG_SMP) || defined(CONFIG_X86_64)
	early_per_cpu(x86_cpu_to_apicid, cpu) = apicid;
	early_per_cpu(x86_bios_cpu_apicid, cpu) = apicid;
#endif
	/* Skip X86_32 code. */

	/* 至此，已确定一个 APIC ID 在 kernel 中的 logical CPU ID.
	 * 首次看到 cpu mask 的 assignment, 所以 cpu mask 其实是 logical CPU ID mask */
	set_cpu_possible(cpu, true);
	physid_set(apicid, phys_cpu_present_map);
	set_cpu_present(cpu, true);
	num_processors++;

	return cpu;
}

/* Parse IOAPIC related entries in MADT returns 0 on success, < 0 on error */
static int __init acpi_parse_madt_ioapic_entries(void)
{
	int count;
	/* Skip 不重要的 code. */

	/* if "noapic" boot option, don't look for IO-APICs */
	/* kernel parameter "noapic" 是指不使用 I/O APIC. 名不副实 so much! */
	if (skip_ioapic_setup) {
		printk(KERN_INFO PREFIX "Skipping IOAPIC probe "
		       "due to 'noapic' option.\n");
		return -ENODEV;
	}

	/* Quick knowledge: 系统中可能有一或多个 I/O APIC, each I/O APIC resides at a
	 * unique address, address 应是指 direct register 中的 index register 的
	 * memory address. 每个 I/O APIC chip 对应一个 I/O APIC Structure.
	 *
	 * Kernel 用 struct ioapic 描述一个 I/O APIC chip, 用 nr_ioapics 表示它的数目。
	 * MADT's I/O APIC structure 的处理在 acpi_parse_ioapic --> mp_register_ioapic,
	 * (很复杂) 该函数主要是初始化相应的 struct ioapic(ioapics[MAX_IO_APICS]).
	 *
	 * NOTE: kernel 用 struct ioapic.nr_registers 描述 I/O APIC's redirection
	 * table 中 redirection register 的数目，并被它为 IRQ routing register.
	 */
	count = acpi_table_parse_madt(ACPI_MADT_TYPE_IO_APIC, acpi_parse_ioapic,
				      MAX_IO_APICS);
	if (!count) {
		printk(KERN_ERR PREFIX "No IOAPIC entries present\n");
		return -ENODEV;
	} else if (count < 0) {
		printk(KERN_ERR PREFIX "Error parsing IOAPIC entry\n");
		return count;
	}

	/* 下面的内容超出了知识边界，待回头继续分析。Go back. */
	count = acpi_table_parse_madt(ACPI_MADT_TYPE_INTERRUPT_OVERRIDE,
				      acpi_parse_int_src_ovr, nr_irqs);
	if (count < 0) {
		printk(KERN_ERR PREFIX
		       "Error parsing interrupt source overrides entry\n");
		/* TBD: Cleanup to allow fallback to MPS */
		return count;
	}

	/*
	 * If BIOS did not supply an INT_SRC_OVR for the SCI
	 * pretend we got one so we can set the SCI flags.
	 * But ignore setting up SCI on hardware reduced platforms.
	 */
	if (acpi_sci_override_gsi == INVALID_ACPI_IRQ && !acpi_gbl_reduced_hardware)
		acpi_sci_ioapic_setup(acpi_gbl_FADT.sci_interrupt, 0, 0,
				      acpi_gbl_FADT.sci_interrupt);

	/* Fill in identity legacy mappings where no override */
	mp_config_acpi_legacy_irqs();

	count = acpi_table_parse_madt(ACPI_MADT_TYPE_NMI_SOURCE,
				      acpi_parse_nmi_src, nr_irqs);
	if (count < 0) {
		printk(KERN_ERR PREFIX "Error parsing NMI SRC entry\n");
		/* TBD: Cleanup to allow fallback to MPS */
		return count;
	}

	return 0;
}
```
至此，ACPI boot phase initialization 告终。后面仍有一些函数：

  - get_smp_config()
  - init_apic_mappings()
  - prefill_possible_map()
  - init_cpu_to_node()

See what they do.

函数 get_smp_config 与更早调用的 find_smp_config 密切相关。find_smp_config 在 memory 中寻找 MPS(MultiProcessor Specification) 的 MP Floating Pointer Structure, 分别有变量 *mpf_base*, *mpf_found* 记录信息。 Generally speaking, MPS 是古老的 spec, 它的功能已完全被 ACPI 包含，所以，正常情况下，二者有其一即可[1]。 但(个人推测), 在使用 MPS 到 ACPI 的演进过程中，出现过二者都有的 case.

[1]. https://en.wikipedia.org/wiki/MultiProcessor_Specification

NUMA 初始化之后来到 get_smp_config, 实际是 default_get_smp_config 函数。目前，其入参 **early** 唯一的使用者是 amd_numa_init(), 为获取 boot_cpu_physical_apicid. 但不得不说，该函数内容与 APCI MADT 处理交织，逻辑让我困惑。

amd_numa_init() 仅用于基于 AMD Operon (K8) CPU 实现的 NUMA system 的初始化，因为其 NUMA 信息存储在 NorthBirdge, 后来的设计才回归正常： 使用 ACPI. Refer: [NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access), [Opteron](https://en.wikipedia.org/wiki/Opteron). Simply speaking: AMD Opteron 是第一款 X86_64 CPU, 且 AMD 基于它实现了第一个 NUMA system.

```
/* 对 "early = 1" 重点分析。*/
void __init default_get_smp_config(unsigned int early)
{
	struct mpf_intel *mpf;

	/* 在 ACPI MADT 的处理中， smp_found_config 和 acpi_lapic 都是同时 set.
	 * 若 smp_found_config = 0, 说明没有 ACPI MADT && 没有 MP table, return.
	 * 若二者有任意一个，则走下去。 */
	if (!smp_found_config)
		return;

	/* mpf_found = 0, 表明无 MP table. 若有 MP table, 走下去。*/
	if (!mpf_found)
		return;

	/* 现在情况有 2： 只有 MP table, 或 2 个 table 都有。*/

    /* -------------------------------------------------------------------------
     * 单独分析 amd_numa_init: early = 1：
     *   1. 若 acpi_lapic = 1(有 ACPI MADT)： 上面 if(!smp_found_config) 就已 return.
     *      ACPI 已由 register_lapic_addr 拿到 boot_cpu_physical_apicid.
     *   2. 若 acpi_lapic = 0(无 ACPI MADT), 走下去，将由 check_physptr -->
     *      smp_read_mpc--> register_lapic_address 同样获取，且及时 return.
     *   		if (!acpi_lapic)
	 *				register_lapic_address(mpc->lapic);
	 *
	 *			if (early)
	 *				return 1;
	 *--------------------------------------------------------------------------
	 *
	 * early = 0 的正常情况下，走下去，情况如上所述只有 2 种。下面只基于 early = 0 继续分析。
	 *
	 * 所以，上面分析了 2 种会继续走下去的情况：
	 *   1. early = 1 && 无 ACPI MADT(acpi_lapic = 0)，后面代码解析 MP table 拿到
	 *      boot CPU ID 返回
	 *   2. early = 0(只有 MP， 或二者都有)
     */
	if (acpi_lapic && early)
		return;

	/* early = 1 在上面已集中分析完，后面单独分析 early = 0.
	 * 所以现在的情况是： early = 0 && (只有 MP table || 2 个 table 都有) */

	/* MPS doesn't support hyperthreading, aka only have thread 0 apic id
	 * in MPS table */
	/* 知识点：MPS 列出的 APIC ID 都是 physical core, 不支持 Intel HyperThread
	 * 技术的 logical core. */

    /* 若 ACPI MADT 提供了 both local APIC & I/O APIC, 则不必 parse MP? 表明 ACPI
     * MADT's I/O APIC entry 足以 cover MP table 中 processor 外其他 entry info.
     * 暂不理解。 */
	if (acpi_lapic && acpi_ioapic)
		return;

	mpf = early_memremap(mpf_base, sizeof(*mpf));
	if (!mpf) {
		pr_err("MPTABLE: error mapping MP table\n");
		return;
	}

	pr_info("Intel MultiProcessor Specification v1.%d\n", mpf->specification);
#if defined(CONFIG_X86_LOCAL_APIC) && defined(CONFIG_X86_32)
	if (mpf->feature2 & (1 << 7)) {
		pr_info("    IMCR and PIC compatibility mode.\n");
		pic_mode = 1;
	} else {
		pr_info("    Virtual Wire compatibility mode.\n");
		pic_mode = 0;
	}
#endif

	/* Now see if we need to read further. */
	/* MP Floating Pointer Structure's feature byte1 表示 MP System Configuration
	 * Type. MPS 预定义了 default MP system configurations, refer: MPS chapter 5.
	 */
	if (mpf->feature1) {
		/* OS 根据 default config 自己构造 MP table. */
		if (early) {
			/* local APIC has default address */
			/* MP default configuration 的条件之一： APIC base address = 0xFEE00000 */
			mp_lapic_addr = APIC_DEFAULT_PHYS_BASE;
			goto out;
		}

		/* 注意：构造的 processor entry 未设置 BP(Boot Processor) flag. 那么后面解析
		 * 时将无法得到 boot_cpu_physical_apicid. */
		pr_info("Default MP configuration #%d\n", mpf->feature1);
		construct_default_ISA_mptable(mpf->feature1);
	} else if (mpf->physptr) {
	    /* 不使用 MPS default MP configuration, 解析 MP table. */
		if (check_physptr(mpf, early))
			goto out;
	} else
		BUG();

	if (!early)
		pr_info("Processors: %d\n", num_processors);

	/* Only use the first configuration found. */
out:
	early_memunmap(mpf, sizeof(*mpf));
}

static int __init check_physptr(struct mpf_intel *mpf, unsigned int early)
{
	struct mpc_table *mpc;
	unsigned long size;

	size = get_mpc_size(mpf->physptr);
	mpc = early_memremap(mpf->physptr, size);

	/* Read the physical hardware table.  Anything here will override the defaults. */
	/* Read MultiProcessor Configuration. */
	if (!smp_read_mpc(mpc, early)) {
#ifdef CONFIG_X86_LOCAL_APIC
		smp_found_config = 0;
#endif
		pr_err("BIOS bug, MP table errors detected!...\n");
		pr_cont("... disabling SMP support. (tell your hw vendor)\n");
		early_memunmap(mpc, size);
		return -1;
	}
	early_memunmap(mpc, size);

	if (early)
		return -1;

#ifdef CONFIG_X86_IO_APIC
	/*
	 * If there are no explicit MP IRQ entries, then we are
	 * broken.  We set up most of the low 16 IO-APIC pins to
	 * ISA defaults and hope it will work.
	 */
	if (!mp_irq_entries) {
		struct mpc_bus bus;

		pr_err("BIOS bug, no explicit IRQ entries, using default mptable. (tell your hw vendor)\n");

		bus.type = MP_BUS;
		bus.busid = 0;
		memcpy(bus.bustype, "ISA   ", 6);
		MP_bus_info(&bus);

		construct_default_ioirq_mptable(0);
	}
#endif

	return 0;
}

/* 返回值是 mapping 的 logical CPU number. */
static int __init smp_read_mpc(struct mpc_table *mpc, unsigned early)
{
	char str[16];
	char oem[10];

    ...

	/* Initialize the lapic mapping */
	/* -------------------------------------------------------------------------
	 * amd_numa_init() --> early_get_smp_config() 在这两个 if() 处结束。early = 1
	 * 时，只有 acpi_lapic = 0 时才会走到这里。
	 * -------------------------------------------------------------------------
	 *
	 * 回忆上文的 assumption: early = 0, 只有 MP table 或二者都有。有 ACPI MADT 的话，
	 * register_lapic_address 已调用过，这里不需重复。
	 *
	 * 若无 ACPI MADT，看起来有个*小问题*：这里 register_lapic_address 中通过读取寄存器
	 * 获得 boot_cpu_physical_apicid, 但下面 MP_processor_info 中又会从 processor
	 * entry 中获取并 override it. */
	if (!acpi_lapic)
		register_lapic_address(mpc->lapic);

	if (early)
		return 1;

    ...

	/* Now process the configuration blocks. */

	/* 能走到下面代码的条件：第 1 种很明显，只有 MP table； 第 2 种情况，2 个 table 都有，
	 * 但 ACPI MADT 提供了 local APIC, 没提供 I/O APIC, 由 MP table 提供。 所以说，
	 * ACPI MADT 的 I/O APIC info cover 了 MP table 种 processor 以外的其他 entry.*/
    ...

	while (count < mpc->length) {
		switch (*mpt) {
		case MP_PROCESSOR:
			/* ACPI may have already provided this data */
			if (!acpi_lapic)
				MP_processor_info((struct mpc_cpu *)mpt);
			skip_entry(&mpt, &count, sizeof(struct mpc_cpu));
			break;
		case MP_BUS:
			...
		case MP_IOAPIC:
			...
		case MP_INTSRC:
			...
		case MP_LINTSRC:
			...
		default:
			/* wrong mptable */
			smp_dump_mptable(mpc, mpt);
			count = mpc->length;
			break;
		}
		x86_init.mpparse.mpc_record(1);
	}

	if (!num_processors)
		pr_err("MPTABLE: no processors registered!\n");
	return num_processors;
}

```
OK, 从 firmware 获取 SMP 架构信息的处理至此应该全结束了, moving on.

初看 init_apic_mappings, 函数名和代码都让人困惑。它的工作看似仅初始化 boot_cpu_physical_apicid, 而正常情况下，前面解析 ACPI MADT 和 MP table 时(分别通过 register_lapic_address & MP_processor_info)会初始化它。唯一的线索落在引入这几行代码的 commit *1e90a13d0c3dc*.

调用处的 comments:

>Systems w/o ACPI and mptables might not have it mapped the local APIC yet, but prefill_possible_map() might need to access it.

一眼看上去也是不知所云，看上去和后面的 prefill_possible_map() 有关。

背景知识：发现系统是 SMP 时，或者说发现 APIC 时，若非 x2APIC mode, 首先要获得 APIC register base address 并映射到虚拟地址空间，方便读写；获取 boot_cpu_physical_apicid. 这些正是函数 register_lapic_address 的内容。

上面提到的正常情况：有 ACPI MADT 或 MP table 时, 背景提到的 2 件事都有做过：

  1. ACPI MADT: register_lapic_address
  2. MP table: smp_read_mpc --> MP_processor_info. 前者 mapping APIC, 后者获取 boot_cpu_physical_apicid.

而这里 init_apic_mappings 其实是为兼容不正常的 corner case, 如其调用处 comments 所述：无 ACPI MADT 和 MP table 时，APIC register base address 没有被 mapping(也没有初始化 boot_cpu_physical_apicid), 具体 case 如 commit 所述：

>left a wreckage for systems which have neither ACPI nor mptables, but the CPU has an APIC, e.g. virtualbox.

函数名终于 make sense, 而且后面的 prefill_possible_map 有调用 read_apic_id, 需要使用 virtual address 访问 memory mapped APIC registers. 若碰到 corner case 没有 map local APIC register, read_apic_id 时肯定要出问题！

**问题**: commit log 中提到的 virtualbox 的例子，无 ACPI MADT & MP table, 却有 APIC, 它是 UP 还是 SMP??? log 没有明确说明，目前只能推测它只可能是 UP, 等待验证。

纵览函数后，列出不正常情况的 **corner case**:

  1. (如 commit 所说:)ACPI MADT & MP table 都没有，却有 APIC. 这时不会调用 register_lapic_address.
  2. 使用 MPS 的 default configuration 时， OS 自己构造的 MP table Processor entry 未设置 BP flag, 也未 mapping APIC register base address.

Tips again: 此函数专为 cover 那些 corner case! 同时也要兼容正常情况！
```
void __init init_apic_mappings(void)
{
	unsigned int new_apicid;

	/* 略过 */
	apic_check_deadline_errata();

	/* 背景：无 ACPI MADT 和 MP table; x2APIC 所有寄存器是 MSR, 不需 memory mapping.
	 * 所以直接 read MSR 获得 boot_cpu_physical_apicid 即可，无需 mapping. */
	if (x2apic_mode) {
		boot_cpu_physical_apicid = read_apic_id();
		return;
	}

	/* If no local APIC can be found return early */
	/* 前者: 无 ACPI MADT & MP table； 后者: CPU 无 APIC, 这情况明显 disable APIC.
	 * 后者的 comments: "Detect and enable local APICs on non-SMP boards", 看来
	 * 早就知道 virtualbox 这种 corner case? */
	if (!smp_found_config && detect_init_APIC()) {
		/* lets NOP'ify apic operations */
		pr_info("APIC: disable apic facility\n");
		apic_disable();
	} else {
		/* 进入此分支的另外 3 个条件：
		 *   1. 无 table, 有 APIC. 正是 commit 中提到的 corner case.
		 *   2. 有 table, 无 APIC, (目前认为)不可能有这种 case
		 *   3. 有 table, 有 APIC, 这是正常情况，此 branch 不处理。
		 *
		 * 看来此分支专为处理条件 1. mp_lapic_addr 在上面 detect_init_APIC 中已赋值。
		 * apic_phys 看起来冗余？ */
		apic_phys = mp_lapic_addr;

		/* If the system has ACPI MADT tables or MP info, the LAPIC
		 * address is already registered.	 */
		/* 条件 1. 从来没有调用过 register_lapic_address, 所以这里处理一下。*/
		if (!acpi_lapic && !smp_found_config)
			register_lapic_address(apic_phys);
	}

	/* Fetch the APIC ID of the BSP in case we have a
	 * default configuration (or the MP table is broken). */

	/* 下面的代码处理另一 corner case: 使用 MPS default configuration.
	 *
	 * Quick knowledge:
	 * MP spec defines several default MP system configurations. The purpose
	 * of these defaults is to simplify BIOS design. If a system conforms to
	 * one of the default configurations, the BIOS will not need to provide
	 * the MP configuration table. The OS will have the default MP configuration
	 * table predefined internally.
	 *
	 * MPS 定义的 default configurations 都只有 2 个 CPU, Linux kernel 构造 MP
	 * table 时 APIC ID is set 0 and 1, 但没有设置 processor entry 中的 BP flag,
	 * 所以未初始化 boot_cpu_physical_apicid.
	 */
	new_apicid = read_apic_id();
	if (boot_cpu_physical_apicid != new_apicid) {
		boot_cpu_physical_apicid = new_apicid;
		/*
		 * yeah -- we lie about apic_version
		 * in case if apic was disabled via boot option
		 * but it's not a problem for SMP compiled kernel
		 * since apic_intr_mode_select is prepared for such
		 * a case and disable smp mode
		 */
		boot_cpu_apic_version = GET_APIC_VERSION(apic_read(APIC_LVR));
	}
}

#ifdef CONFIG_X86_64
/*
 * Detect and enable local APICs on non-SMP boards.
 * Original code written by Keir Fraser.
 * On AMD64 we trust the BIOS - if it says no APIC it is likely
 * not correctly set up (usually the APIC timer won't work etc.)
 */
/* 此函数仅被 init_apic_mapping 调用，且仅在 !smp_found_config = TRUE 时调用。 */
static int __init detect_init_APIC(void)
{
	if (!boot_cpu_has(X86_FEATURE_APIC)) {
		pr_info("No local APIC present\n");
		return -1;
	}

	/* 所以这里的赋值仅为： 无 table, 有 APIC 的 case. */
	mp_lapic_addr = APIC_DEFAULT_PHYS_BASE;
	return 0;
}
#else

#endif
```
prefill_possible_map() 紧挨着 init_apic_mappings(). 这里涉及 2016 年 10 月的 3 个 commit, 我们来还原下历史：

  1. 10/3: 2a51fe083eba7 解决 kdump kernel 碰巧运行在 hotplugged CPU 上，导致该 CPU 没有调用 generic_processor_info() 映射 kernel logical CPU 的问题。
  2. 10/22: ff8560512b8d4 发现 1. 会导致无 ACPI & MP table & APIC 的普通 UP doesn't boot.
  3. 10/29: 1e90a13d0c3dc 发现 2. 会导致 virtualbox 这种 corner case(UP, 无 APCI
     MADT & MP table, BUT has APIC) crash.

In other words, 当系统只有 1 个 CPU 时，可能有几种情况：

  1. SMP, kdump kernel is running.
  2. 普通 UP
  3. Corner case UP: virtualbox

```
/*
 * cpu_possible_mask should be static, it cannot change as cpu's are onlined,
 * or offlined. The reason is per-cpu data-structures are allocated by some
 * modules at init time, and don't expect to do this dynamically on CPU's
 * arrival/departure.
 * cpu_present_mask on the other hand can change dynamically.
 * In case when cpu_hotplug is not compiled, then we resort to current
 * behaviour, which is cpu_possible == cpu_present.
 * - Ashok Raj
 *
 * Three ways to find out the number of additional hotplug CPUs:
 * - If the BIOS specified disabled CPUs in ACPI/mptables use that.
 * - The user can overwrite it with possible_cpus=NUM
 * - Otherwise don't reserve additional CPUs.
 * We do this because additional CPUs waste a lot of memory.
 * -AK
 */
/* "possible CPU" 指一个计算机系统实际支持的最大 CPU 数量，当支持 CPU hotplug 时，它
 * 包括已 onlined, offlined(待 hotplug CPU); 不支持时则等于 onlined CPU.
 * 由上面 comments 可知，percpu 变量根据系统实际支持最大 CPU 数分配空间，尤其部分
 * module 在 booting 阶段就要初始化 percpu 变量。
 * 系统支持 CPU hotplug 时， present != online, present 可能仅是物理上的插上。
 *
 * 本函数要确定 possible CPU 数目. */
__init void prefill_possible_map(void)
{
	int i, possible;

	/* No boot processor was found in mptable or ACPI MADT */
	/* 变量 num_processors 在 generic_processor_info() 中初始化，表示从 ACPI MADT
	 * 或者 MP table 中 enumerated *enabled CPU* 数. cpu_possible_mask &
	 * cpu_present_mask 都根据它来初始化。
	 *
	 * The cases of num_processors = 0, according to 2a51fe083eba7, there
	 * are 3:
	 *   1. 有 APIC 却无 ACPI MADT & MP table, 即 1e90a13d0c3dc 所述 virtualbox
	 *      case, 只执行一次 generic_processor_info, 即 number_processors = 1,
	 *      间接验证该 commit 描述的 corner case 就是 UP;
	 *   2. 如 commit 所述: kdump 时，hot-plugged CPU 是 boot CPU. (IIRC, Kdump
	 *      kernel 只使用一个 CPU 运行，即 nr_cpu_id=1). 这时 kdump kernel 依然
	 *      在 early_acpi_boot_init --> register_lapic_addr 通过读取 register
	 *      获得 boot_cpu_physical_apicid. 但 acpi_boot_init 解析 MADT, enumerate
	 *      CPU 时 无法找到 boot CPU, 所以 num_processors 为 0.
	 *   3. 普通 UP, 无 ACPI & MP table & APIC, 直接 num_processors = 0, 无需
	 *      map kernel logical CPU.
	 */
	if (!num_processors) {
		if (boot_cpu_has(X86_FEATURE_APIC)) {
			/* case 1 & 2.  变量 cpu 仅用来 debug, 但它和 apicid 好像是一回事?
			 * No matter what case, boot_cpu_physical_apicid 都会被初始化, case 1
			 * 在 init_apic_mapping(), case 2 的 kdump kernel 依然在解析 MADT
			 * header 时初始化它。*/
			int apicid = boot_cpu_physical_apicid;
			int cpu = hard_smp_processor_id();

			/* case 1 无 table; case 2 hotplug(boot) CPU 不在 ACPI MADT 中；
			 * table is provide by BIOS. so, they are "not listed by BIOS". */
			pr_warn("Boot CPU (id %d) not listed by BIOS\n", cpu);

			/* Make sure boot cpu is enumerated */
			if (apic->cpu_present_to_apicid(0) == BAD_APICID &&
			    apic->apic_id_valid(apicid))
				generic_processor_info(apicid, boot_cpu_apic_version);
		}

		/* case 3. */
		if (!num_processors)
			num_processors = 1;
	}

	/* Kernel parameter "maxcpus"(setup_max_cpus) 表示 booting 阶段能 bringup
	 * 的最大 CPU 数。未 bringup 的 CPU, 可在启动后使其 online. 0 表示 "nosmp".
	 * 使用场景? 系统提供 8 个 CPU，但某次只想使用 2 个，若还需要，则开机后动态 online?
	 * 理解它很重要，对理解代码逻辑很重要。*/
	i = setup_max_cpus ?: 1;
	/* 若未设置 possible_cpus in kernel parameter, 则计算 possible cpu 的值；
	 * 若设置，则使用它. */
	if (setup_possible_cpus == -1) {
		/* num_processors 是已 enumerated 的 online(enabled) CPU, 作为 base. */
		possible = num_processors;

#ifdef CONFIG_HOTPLUG_CPU
		/* disabled_cpus 包括：MADT 中 flags 是 disabled 的，以及因内核最大支持数限制
		 * (nr_cpus)而被归入 disabled 的。支持 CPU hotplug 时，他们都是 possible CPU. */
		if (setup_max_cpus)
			possible += disabled_cpus;
#else
		/* 不支持 hotplug 时，possible CPU = num_processors, 而且不能超过 maxcpus. */
		if (possible > i)
			possible = i;
#endif
	} else
		possible = setup_possible_cpus;

	/* 正常情况下，二者应相等。异常情况是？ */
	total_cpus = max_t(int, possible, num_processors + disabled_cpus);

	/* nr_cpu_ids could be reduced via nr_cpus= */
	if (possible > nr_cpu_ids) {
		pr_warn("%d Processors exceeds NR_CPUS limit of %u\n",
			possible, nr_cpu_ids);
		possible = nr_cpu_ids;
	}

#ifdef CONFIG_HOTPLUG_CPU
	if (!setup_max_cpus)
#endif
	if (possible > i) {
		pr_warn("%d Processors exceeds max_cpus limit of %u\n",
			possible, setup_max_cpus);
		possible = i;
	}

	/* 修改 nr_cpu_ids 为系统实际支持数目 */
	nr_cpu_ids = possible;

	pr_info("Allowing %d CPUs, %d hotplug CPUs\n",
		possible, max_t(int, possible - num_processors, 0));

	reset_cpu_possible_mask();

	/* 之前 generic_processor_info 中只 set enabled CPU into possible CPU,
	 * 现在计入了 disabled CPU, 即 hotpluggable CPU. */
	for (i = 0; i < possible; i++)
		set_cpu_possible(i, true);
}
```
kernel parameter *maxcpus* & *possible_cpus* 的[简单介绍](https://www.ibm.com/support/knowledgecenter/en/linuxonibm/com.ibm.linux.z.lgdd/lgdd_r_maxcpusparm.html)

prefill_possible_map() 后紧跟着 init_cpu_to_node(), 现在可以回到上面 NUMA Initialization 一节。