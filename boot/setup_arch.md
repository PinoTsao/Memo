# x86 setup_arch

One can imagine, some OS sub-systems depend on architecture support to function. **setup_arch** function is used for what the name tells. It involves different areas of an architecture, for x86, mainly CPU & memory.

Tips: according to [Intel 5-level paging white paper](https://software.intel.com/content/dam/develop/public/us/en/documents/5-level-paging-white-paper.pdf), *1.1 Existing Paging in IA-32e Mode*:

  - before 5-level paging, Intel 64 bit processor supports **at most** 48-bit linear(virtual) address, and 46-bit physical address respectively;
  - after 5-level paging, it supports **at most** 57-bit linear address, and 52-bit physical address respectively.

According to tips above, linear(virtual) address bit increase has a pattern: x86_32 processor only support 32-bit linear address, while 64-bit processor support 48(32 + 16) bits. Due to 64-bit specific canonical address limitation, 64-bit CPU actually use 47(9 + 9 + 9 + 12) bits for virtual address. 5-level paging need one more level paging structure, i.e., need another 9-bit from virtual address, that is 57(48 + 9) bits, limited by canonical address again, 5-level paging actually use 56(9 + 9 + 9 + 9 + 12) bits. 而物理地址增长未发现规律性，仅有 max supported bit 之说。Refer to __VIRTUAL_MASK_SHIFT &  MAX_PHYSMEM_BITS respectively in code.

```c
/*
 * Determine if we were loaded by an EFI loader.  If so, then we have also been
 * passed the efi memmap, systab, etc., so we should use these data structures
 * for initialization.  Note, the efi init code path is determined by the
 * global efi_enabled. This allows the same kernel image to be used on existing
 * systems (with a traditional BIOS) as well as on EFI systems.
 */
/*
 * setup_arch - architecture-specific boot-time initializations
 *
 * Note: On x86_64, fixmaps are ready for use even before this is called.
 */

void __init setup_arch(char **cmdline_p)
{
	/* Detail analysis of __pa_symbol in a separate section below.
	 * Tip: Two "_text" are of absolute addressing, i.e., S + A.
	 * First appearance of memblock_reserve.
	 */
	/* The following memblock reservation work is going to converge into a single
	 * function via commit a799c2bd29d19c565f.
	 */
	/*
	 * Reserve the memory occupied by the kernel between _text and
	 * __end_of_kernel_reserve symbols. Any kernel sections after the
	 * __end_of_kernel_reserve symbol must be explicitly reserved with a
	 * separate memblock_reserve() or they will be discarded.
	 */
	memblock_reserve(__pa_symbol(_text),
			 (unsigned long)__end_of_kernel_reserve - (unsigned long)_text);

	/*
	 * Make sure page 0 is always reserved because on systems with
	 * L1TF its contents can be leaked to user processes.
	 */
	memblock_reserve(0, PAGE_SIZE);

	/* There is reserve_initrd() later, difference? To be compared. */
	early_reserve_initrd();

	/*
	 * At this point everything still needed from the boot loader
	 * or BIOS or kernel text should be early reserved or marked not
	 * RAM in e820. All other memory is free game.
	 */

#ifdef CONFIG_X86_32
	...
#else
	/* copy_bootdata() initialized boot_command_line. */
	printk(KERN_INFO "Command line: %s\n", boot_command_line);
	/* MAX_PHYSMEM_BITS = 46, i.e., 64TB. Refer to tips at the top of this article. */
	boot_cpu_data.x86_phys_bits = MAX_PHYSMEM_BITS;
#endif

	/*
	 * If we have OLPC OFW, we might end up relocating the fixmap due to
	 * reserve_top(), so do this before touching the ioremap area.
	 */
	/* 新概念 get: One Laptop Per Child. Open FirmWare. Omit. */
	olpc_ofw_detect();

	/* Install #DB & #BP handler into IDT.
	 * Is reloading IDTR necessary after modify IDT? Seems so
	 */
	idt_setup_early_traps();

	/* Main job: Initialize boot_cpu_data via early_identify_cpu(). Details at below. */
	early_cpu_init();
	...
	/* 新概念 get: https://lwn.net/Articles/412072/ */
	jump_label_init();

	/* A new security feature. TBD. */
	static_call_init();

	/* During kernel initialization: CPU often need one-time access of certain memory,
	 * like I/O device for setting up, data from firmware. They are mapped into
	 * temporary area called fixmap for the purpose. Details at below.
	 */
	early_ioremap_init();
	...

	/* x86_init is defined in arch/x86/kernel/x86_init.c, with a set of default
	 * functions for setting up standard PC hardware. For specific platforms,
	 * such as Intel MID, Xen, they substitute some default functions with ones
	 * of themselves, or complement the default one.
	 *
	 * Specific OEM products may have their specific platform setup. In our case,
	 * it defaults to x86_init_noop() and does nothing.
	 */
	x86_init.oem.arch_setup();

	/* Resource is managed by struct resource, which is tree-like structure,
	 * indicating a range of address of I/O ports, physical address.
	 * x86_phys_bits is obtained via CPUID, the expression is self-documented.
	 */
	iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;

	/* Both following 2 functions are for get E820 data.
	 *
	 * Put boot_params.e820_table into kernel's e820_table and sanitize:
	 * sorting, overlap removing.
	 */
	e820__memory_setup();

	/* The concept of "setup data" is of boot protocol, which is used by firmware
	 * to provide more info/data to Linux kernel. "Setup data" provides extended
	 * E820 info, get it into kernel's e820_table if there is.
	 *
	 * It is estimated that NUMA has excessive E820 info entry, while ordinary PC
	 * doesn't.
	 * Tip: the boot_params currently used is kernel data, which is already reserved
	 * at the entry of setup_arch, while the setup data doesn't belong to kernel,
	 * which in turn should be reserved explicitly.
	 *
	 * fixmap area is just initialized, now is used via early_memremap &
	 * early_memunmap.  Tip: mapped via level2_fixmap_pgt 505th PMD entry.
	 */
	parse_setup_data();

	...

	/* 暂时不知为何 2 个 command line buffer. */
	strlcpy(command_line, boot_command_line, COMMAND_LINE_SIZE);
	*cmdline_p = command_line;
	...

	/* Later in start_kernel, will be called again. A local static symbol prevent
	 * it from being called again. "Early parsing" parses the parameters only with
	 * struct obs_kernel_param.early set & a special one "console", which is
	 * implemented as "earlycon" in code.
	 *
	 * Parsing string pattern "foo=bar,bar2 baz=fuz wiz" to get param & value,
	 * feed the pair directly to do_early_param(). Details at below.
	 */
	parse_early_param();
	...

#ifdef CONFIG_MEMORY_HOTPLUG
	/* Need to understand memblock_set_bottom_up(). Why set allocation order would
	 * make sure later allocation are around the memory of Linux kernel itself?
	 * TBD. */
#endif

	/* The "early param "comments below is from 28bb22379513c, maybe outdated,
	 * it has printk inside at that time, now is gone.
	 *
	 * Could "memblock reserve of setup data" be put into parse_setup_data to
	 * save remap/unmap? TBD.
	 *
	 * A trick inside: input size of early_memremap() is sizeof(struct setup_data),
	 * which doesn't include the size of the data field, but dealing with SETUP_INDIRECT
	 * would access the data with panic, WHY? Early remap's granularity is page(4K),
	 * and it will align the start physical address down & size up to 4K. The range
	 * of one setup data would have two cases:
	 *   1. within one physical page
	 *   2. across the boundary of physical pages
	 *
	 * In both case, the whole range(header & data) of one setup data will be
	 * mapped, this is implicit.
	 */
	/* after early param, so could get panic from serial */
	memblock_x86_reserve_range_setup_data();

	if (acpi_mps_check()) {
		...
	}

	/* Inferring: E820 info originates from firmware, boot loader works as conduit
	 * to pass it to Linux kernel, so boot loader (theoretically) should not taint
	 * it, but add to setup data if it really want to do it.
	 *
	 * E820 data is already extracted from boot_params, but as said above, boot
	 * protocol's setup data doesn't belong to Linux kernel image, and the space it
	 * occupied might be labeled as available(E820_TYPE_RAM) by firmware, because
	 * surely boot loader will choose available memory to construct setup data.
	 *
	 * Update e820_table_kexec & e820_table: Replace E820_TYPE_RAM with
	 * (Linux kernel defined) E820_TYPE_RESERVED_KERN for setup data.
	 */
	e820__reserve_setup_data();

	/* Done with E820: extracted E820 data from boot_params; parsed "mem=" if
	 * command line has it, which sets the max memory size kernel will use,
	 * and removed range beyond max size from E820(but not sanitize it).
	 *
	 * So, sanitize e820_table if has "mem=".
	 */
	e820__finish_early_params();

	/* Refer to System Management BIOS (SMBIOS) Specification at
	 *     https://www.dmtf.org/standards/smbios
	 *
	 * Quick knowledge:
	 * SMBIOS information has 2 parts： Entry Point Structure(EPS) & Structure
	 * Table, EPS is actually a header. Take non-EFI for example, EPS is placed
	 * within physical address range of [0xF0000, 0xFFFFF], and aligned to 16-byte;
	 * Structure Table originates Desktop Management Interface(DMI,obsolete spec,
	 * merged into SMBIOS).
	 *
	 * dmi_setup() remap the physical range above (to get virtual address) to access
	 * the info inside, particularly, get Structure Table physical range (dmi_base
	 * & dmi_len) from EPS & do further parse.
	 *
	 * Some parsed DMI info(such as dmi_system_id) will be outputted to console,
	 * example(folded manually):
	 *   [    0.000000] SMBIOS 2.8 present.
	 *   [    0.000000] DMI: QEMU Standard PC (i440FX + PIIX, 1996), BIOS \
	 *                  rel-1.12.1-0-ga5cab58e9a3f-prebuilt.qemu.org 04/01/2014
	 */
	dmi_setup();

	/*
	 * VMware detection requires dmi to be available, so this
	 * needs to be done after dmi_setup(), for the boot CPU.
	 */
	/* interesting things emerge, details at below. */
	init_hypervisor_platform();

	/* TBD. */
	tsc_early_init();
	/* callback function name also is: probe_roms(). Details at below. */
	x86_init.resources.probe_roms();
	...

	/* Make sure the range of kernel itself is marked with E820_TYPE_RAM in kernel's
	 * e820_table.  WHY? */
	e820_add_kernel_range();

	/* Another kind of sanitize, trim is also a it is well-documented inside. For memory
	 * map info, refer to: https://wiki.osdev.org/Memory_Map_(x86) */
	trim_bios_range();

	/* https://en.wikipedia.org/wiki/Graphics_address_remapping_table. TBD */
	early_gart_iommu_check();

	/*
	 * partially used pages are not usable - thus we are rounding upwards:
	 */
	/* Background: pfn = page frame number. Physical memory(all type, RAM, ROM, etc)
	 * is divided into frames by page size, frames are indexed from 0, so pfn range
	 * would be [0 - max].
	 *
	 * max_pfn denotes the max PFN of RAM(E820_TYPE_RAM) in e820_table, but memory
	 * may has holes.
	 */
	max_pfn = e820__end_of_ram_pfn();

	/* update e820 for memory not covered by WB MTRRs */
	/* Many comments above & inside the function implies: sane BIOS should mark
	 * E820_TYPE_RAM with memory type Write-Back in MTRR(WB provides the best
	 * performance); or else, remove non-WB range from E820_TYPE_RAM, thus,
	 * max_pfn need to be re-calculated.
	 */
	mtrr_bp_init();
	if (mtrr_trim_uncached_memory(max_pfn))
		max_pfn = e820__end_of_ram_pfn();

	/* so, max_pfn is just possible one? max_possible_pfn will be re-evaluate during
	 * NUMA initialization. */
	max_possible_pfn = max_pfn;

	/*
	 * This call is required when the CPU does not support PAT. If
	 * mtrr_bp_init() invoked it already via pat_init() the call has no
	 * effect.
	 */
	/* Emulate? How does it work if PAT is not supported? TBD. */
	init_cache_modes();

	/* Define random base addresses for memory sections after max_pfn is
	 * defined and before each memory section base is used.	*/
	/* TBD */
	kernel_randomize_memory();
	...

#ifdef CONFIG_X86_32
	...
#else
	/* X2APIC enabling depends on 3 prerequisites:
	 *      1. Kernel configuration.
	 * 		2. CPU support X2APIC or not.
	 * 		3. if CPU support, is it enabled or not in MSR.
	 */
	check_x2apic();

	/* 看起来上面 MTRR 的处理把作者 Yinghai Lu 也要搞崩溃了:D
	/* if max_pfn > 4G, max_low_pfn denotes the max PFN of E820_TYPE_RAM under 4G. */
	/* How many end-of-memory variables you have, grandma! */
	/* need this before calling reserve_initrd */
	if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
		max_low_pfn = e820__end_of_low_ram_pfn();
	else
		max_low_pfn = max_pfn;
	/* self-documented. */
	high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
#endif


	/* Find and reserve possible boot-time SMP configuration. */
	/* If configured with CONFIG_X86_MPPARSE, goes to default_find_smp_config().
	 * Refer to Intel Multi-Processor Spec, an outdated spec whose latest version
	 * is 1.4 of 1997, who has been obsoleted by ACPI.
	 * Quick knowledge: MP Floating Pointer Structure locates within first 1M.
	 */
	find_smp_config();

	/* iSCSI, skip. */
	reserve_ibft_region();

	/* details at below. */
	early_alloc_pgt_buf();
	/* Need to conclude brk, before e820__memblock_setup() it could use
	 * memblock_find_in_range, could overlap with brk area. */
	reserve_brk();

	/* Mentioned by comments of __startup_64(). Details at below. */
	cleanup_highmap();

    /* The following allocation from memblock: e820__memblock_alloc_reserved_mpc_new(),
	 * will be limited within 1M.
	 */
	memblock_set_current_limit(ISA_END_ADDRESS);
	/* Put E820_TYPE_RAM & E820_TYPE_RESERVED_KERN regions from e820_table into
	 * memblock.memory, then align all the regions to 4k(start address & size) via
	 * memblock_trim_memory.
	 */
	e820__memblock_setup();

	/* Memory encryption related, Skip. */
	sev_setup_arch();

    /* Acquaintance! Refer to: find_trampoline_placement().
	 * Add BIOS region to memblock.reserved. */
	reserve_bios_regions();

	/* Skip a bunch of efi related functions for now. */

	/* preallocate 4k for mptable mpc */
	/* mpc = multi-processor configuration. If mptable is supported, pre-allocate
	 * memory thru memblock, in order to move it later. Purpose? Looks it is
	 * for kexec according to comments of e820__memblock_alloc_reserved().
	 * Remember: this allocation is limited within 1M. 4K is default, it could
	 * be altered via parameter "alloc_mptable" which is only available when
	 * kernel is configured with CONFIG_X86_MPPARSE.
	 */
	e820__memblock_alloc_reserved_mpc_new();

#ifdef CONFIG_X86_CHECK_BIOS_CORRUPTION
    /* Also depends on CONFIG_X86_BOOTPARAM_MEMORY_CORRUPTION_CHECK, kernel
	 * parameter "memory_corruption_check" & "corruption_check_size". TBD.
	 */
	setup_bios_corruption_check();
#endif

    /* A new world: real mode trampoline, a.k.a x86 trampoline, whose code is
	 * under arch/x86/realmode/. http://lastweek.io/lego/kernel/trampoline gives
	 * a brief introduction.
	 *
     * Real mode trampoline is for BSP to wake up APs in a SMP system, put them
     * into work.
     *
     * It is a separate code image which is included in to VO via directive incbin.
     * Here just allocate memory for it under 1M, and reserve it first.
     * "x86 trampoline": TBD.
     */
	reserve_real_mode();

    /* Workaround for specific platform: certain ranges should not be used by
	 * kernel, reserve them in memblock. Intel SandyBridge only, skip.
	 */
	trim_platform_memory_ranges();
	/* Reserve low memory in memblock.reserved.
	 * Refer to X86_RESERVE_LOW in arch/x86/Kconfig.
	 */
	trim_low_memory_range();

    /* Direct page mapping all available RAM range which is managed by memblock.memory,
	 * starting from PAGE_OFFSET(0xffff888000000000 under 4-level page mapping), using
	 * init_top_pgt, and finally switch to it, early_top_pgt is used before.
	 * Details at below.
	 */
	init_mem_mapping();

	/* IDT was initialized in idt_setup_early_handler() before start_kernel, and
	 * was installed page fault handler only which does page mapping via early_top_pgt,
	 * since CR3 is replaced with init_top_pgt, it make sense to install a new
	 * page fault handler.
	 *
	 * Details at below.
	 */
	idt_setup_early_pf();

	/* mmu_cr4_features is initialized in init_mem_mapping() -> probe_page_size_mask().
	 * Get back to it later.
	 */
	/*
	 * Update mmu_cr4_features (and, indirectly, trampoline_cr4_features)
	 * with the current CR4 value.  This may not be necessary, but
	 * auditing all the early-boot CR4 manipulation would be needed to
	 * rule it out.
	 *
	 * Mask off features that don't work outside long mode (just
	 * PCIDE for now).
	 */
	mmu_cr4_features = __read_cr4() & ~X86_CR4_PCIDE;

	/* Used to be set to 1M, since direct page mapping has been done, it seems
	 * memblock can be fully utilized with max_pfn_mapped.
	 */
	memblock_set_current_limit(get_max_mapped());

	/* printk deserves a big separate chapter, TBD. */
	/* Allocate bigger log buffer. */
	setup_log_buf(1);

	if (efi_enabled(EFI_BOOT)) {
		/* SKIP... */
	}

	/* Details at below. */
	reserve_initrd();

	/* Documentation/admin-guide/acpi/initrd_table_override.rst has the introduction.
	 * Understanding cpio archive format via `man 5 cpio` is necessary for understanding
	 * find_cpio_data().
	 *
	 * It start to make sense why it locates before acpi_boot_table_init().
	 */
	acpi_table_upgrade();

	/* Initialization for specific x86 vendor: ScaleMP(similiar to domestic 浪潮).
	 *
	 * Quick knowledge:
	 * ScaleMP developed the versatile SMP (vSMP) architecture, a unique approach
	 * that enables server vendors to create industry-standard, high-end x86-based
	 * symmetric multiprocessor (SMP) systems.
	 */
	vsmp_init();

	/* TBD */
	io_delay_init();

	/* Confirm whether we are running Apple through DMI. */
	early_platform_quirks();

	/* There are several "acpi" "init" functions which are confusing at first glance,
	 * but will be unraveled after analyzing them all.
	 *
	 * Initialize ACPI table in boot process.
	 */
	/* Parse the ACPI tables for possible boot-time SMP configuration. */
	acpi_boot_table_init();

	/* Different function name style，"acpi_boot_early_init" might be appropriate.
	 * There is acpi_boot_init() later, so the function name can be interpreted as:
	 * early ACPI initialization during boot process.
	 */
	early_acpi_boot_init();

	/* NUMA initialization, NUMA information derive from ACPI tables.
	 * details at below. */
	initmem_init();

	/* Contiguous memory allocator, refer to its file header comments for a quick
	 * knowledge. Details TBD.
	 * Simply, reserve a contiguous chunk of memory via memblock, internal initialization.
	 */
	dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT);

	/* Huge page mixed with CMA? TBD. */
	if (boot_cpu_has(X86_FEATURE_GBPAGES))
		hugetlb_cma_reserve(PUD_SHIFT - PAGE_SHIFT);

	/*
	 * Reserve memory for crash kernel after SRAT is parsed so that it
	 * won't consume hotpluggable memory.
	 */
	/* Just reserve a range of memory from memblock for crash kernel. */
	reserve_crashkernel();

	/* In the first 16M physical address, find RAM size(because there is memory hold)
	 * first, then find free RAM size, the delta would be memory reserved by someone,
	 * set the delta to dma_reserve.
	 */
	memblock_find_dma_reserve();

	if (!early_xdbc_setup_hardware())
		early_xdbc_register_console();

	/* It is paging_init() in init_64.c under X86_64. Details at below. */
	x86_init.paging.pagetable_init();

	kasan_init();

	/* Sync back kernel address range.
	 *
	 * FIXME: Can the later sync in setup_cpu_entry_areas() replace
	 * this call?	X86_32 限定，可略过。 */
	sync_initial_page_table();

	/* Intel's Trusted Execution Technology(TXT), Spec 篇幅不小，但此处初始化比较
	 * 简单: TXT 相关数据通过 boot_params.tboot_addr 传给 kernel, kernel 将其 fixmap
	 * 到预留的 FIX_TBOOT_BASE, then, 检查数据。*/
	tboot_probe();

	/* fixmap 预留给 vsyscall 的虚拟地址空间也 map 起来：__vsyscall_page */
	map_vsyscall();

	early_quirks();

	/* Read APIC and some other early information from ACPI tables. 下文详细分析。*/
	acpi_boot_init();

	/* Intel's Simple Firmware Interface, as a lightweight method for firmware
	 * to export static tables to OS. 用于 Intel MID's Moorestown platform. */
	sfi_init();

	/* Open Firmware 支持 device tree. OF 也是定义 firmware interface 的标准，最初
	 * 用于 non-x86 platform. 看起来现在也可以用在 x86 上了。 NOT interested. */
	x86_dtb_init();

	/* get boot-time SMP configuration. 之前已 find 过，若有 MPTable, 则 remap
	 * 访问 table(细节略). 使用结束则 unmap. */
	get_smp_config();

	/* Systems w/o ACPI and mptables might not have it mapped the local
	 * APIC yet, but prefill_possible_map() might need to access it. */
	/* 看起来只是确认拿到 APIC 地址，并确认 BSP's APIC ID & version. 拿到 APIC 的地址
	 * 就可以 remap 到 fixmap 虚拟地址空间访问。
	 * Tip Again: APIC 地址最终是记录在 mp_lapic_addr 中； APIC ID 是 firmware
	 * report, 可能是不连续的！一个有趣的 macro: MAX_LOCAL_APIC, 实际指的是 ID, 不是
	 * 数量。关于它的讨论：
	 * http://lkml.iu.edu/hypermail/linux/kernel/1509.3/01126.html */
	init_apic_mappings();

	/* 其中的逻辑也是看起来很复杂。不支持 CPU hotplug 时，possible CPU 就是 ACPI MADT
	 * 中所有 enabled CPU; 支持 CPU hotplug 时，那些 hot-pluggable CPU 的 slot 在
	 * ACPI 中已被 mark as disabled CPU, 这时所有的 slot 数量表示 possible CPU. 再
	 * 加上 kernel 自身对 CPU 数的限制(CONFIG & kernel parameter), 经过复杂逻辑，才
	 * 得到真正 possible CPU, set into cpu mask. 待详细分析。*/
	prefill_possible_map();

	/* 涉及 NUMA, percpu, 待需要时再来分析。*/
	init_cpu_to_node();

	/* 科普 again: I/O APIC 有 2 - 3 个 "memory mapped" 32-bit register, 以及一堆
	 * memory indexed register. 前者是 index(仅使用8-bit表示 index), data, EOI(仅在
	 * level-trigger 模式下生效). 后者是 I/O APIC ID & version register, 以及 24 个
	 * interrupt redirection register. I/O APIC 位于南桥 chipset.
	 * 在 acpi_process_madt -..-> mp_register_ioapic 中已解析 I/O APIC entry 并
	 * fixmap 其物理地址，为何这里 memblock 分配一个 page, 重新 fixmap??? 待分析～ */
	io_apic_init_mappings();

	x86_init.hyper.guest_late_init();

	/* Resource tree 相关，待分析。*/
	e820__reserve_resources();
	e820__register_nosave_regions(max_pfn);

	x86_init.resources.reserve_resources();

	/* Background: 分配给 main memory(RAM) 的 CPU 地址空间并不是连续的。
	 * Non-RAM 的 range 被称为 memory hole 或者这里的 gap. 在低 4G 空间中找一块 4M
	 * 的 gap, 用作 PCI MMIO, 记在 pci_mem_start 中。 */
	e820__setup_pci_gap();

#ifdef CONFIG_VT
	...
#endif

	/* default to default_banner() in paravirt.c. On my PC, it prints:
	 *   "Booting paravirtualized kernel on bare hardware"
	 * paravirt, an interesting topic to be learned. */
	x86_init.oem.banner();

	/* Only make sense to devicetree configured system. Skip. */
	x86_init.timers.wallclock_init();

	/* Refer Intel SDM CHAPTER 15 MACHINE-CHECK ARCHITECTURE. TBD. */
	mcheck_init();

	/* 看起来是 1st 时间相关的初始化。*/
	register_refined_jiffies(CLOCK_TICK_RATE);
	...

}
```

## __pa_symbol

Defined in arch/x86/include/asm/page.h. W/o KASLR 时，代码容易理解，不赘述；分析 w/ KASLR 的情况。

Expand all macros related to __pa_symbol():

```c
/* __pa_symbol should be used for C visible symbols.
   This seems to be the official gcc blessed way to do such arithmetic. */
/*
 * We need __phys_reloc_hide() here because gcc may assume that there is no
 * overflow during __pa() calculation and can optimize it unexpectedly.
 * Newer versions of gcc provide -fno-strict-overflow switch to handle this
 * case properly. Once all supported versions of gcc understand it, we can
 * remove this Voodoo magic stuff. (i.e. once gcc3.x is deprecated)
 */
#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide((unsigned long)(x)))

/* arch/x86/include/asm/page_64.h */
#define __phys_reloc_hide(x)	(x)

#define __phys_addr_symbol(x) \
	((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

`readelf -r setup.o` 可知 _text 是绝对地址寻址，so, under KASLR, ZO 会加上 v_delta.
 Fully expanded __phys_addr_symbol is:

>VMA + v_delta - __START_KERNEL_map + p_delta - v_delta = VMA - __START_KERNEL_map + p_delta = LMA + p_delta = 符号的实际物理地址

The little trick is: VMA - __START_KERNEL_map = LMA.

Take a step back, 假设使用 **__pa_symbol** 的符号是 PC relative 寻址，入参是符号的实际虚拟地址，即 LMA + v_delta, 逻辑相同。

## early_cpu_init

**__x86_cpu_dev_start** & **__x86_cpu_dev_end** are symbols defined in linker script, used for marking the boundary of a special section, which is used as following:

```c
#define cpu_dev_register(cpu_devX) \
	static const struct cpu_dev *const __cpu_dev_##cpu_devX __used \
	__attribute__((__section__(".x86_cpu_dev.init"))) = \
	&cpu_devX;
```

不同 vendor 的 x86 CPU 设计不同，有各自的特殊数据和设置，他们被抽象为 struct cpu_dev. 上面的 macro 定义比较 smart, 只定义 pointer 指向 struct cpu_dev. 在 early_cpu_init 中则很容易访问相应的 struct cpu_dev.

early_cpu_init 的主要作用是初始化 *boot_cpu_data* in early_identify_cpu(), especially *boot_cpu_data.x86_capability[]*. 该函数大量使用 CPUID 指令，refer to its description in Intel SDM 2.

Linux kernel 中调用 CPUID 指令的实现涉及 paravirt 机制，interesting, TBD. 目前的了解：某些指令和操作难以虚拟化，所以使用 paravirt 机制进行虚拟化。paravirt.c 中定义了需要 paravirt 的操作： struct paravirt_patch_template pv_ops, 各虚拟化平台(KVM, XEN, Vmware, etc)自行替换其中的 hook funtion. 还定义了 struct pv_info pv_info.

```c
/*
 * Do minimum CPU detection early.
 * Fields really needed: vendor, cpuid_level, family, model, mask,
 * cache alignment.
 * The others are not touched to avoid unwanted side effects.
 *
 * WARNING: this function is only called on the boot CPU.  Don't add code
 * here that is supposed to run on all CPUs.
 */
static void __init early_identify_cpu(struct cpuinfo_x86 *c)
{
#ifdef CONFIG_X86_64
	c->x86_clflush_size = 64;
	/* x86_64's physical address bit width is unpredictable, this seems to be the
	 * not-making-too-much-sense initialization merely, the following get_cpu_address_sizes
	 * will get the real width via CPUID.
	 * (physical address bit width of x86_32 w/ PAE is 36.)
	 */
	c->x86_phys_bits = 36;
	c->x86_virt_bits = 48; /* Refer to tips at the top of this article. */
#else
	...
#endif

	/* level is the leaf number in CPUID parlance. */
	c->extended_cpuid_level = 0;

	/* X86_64 defaults to support CPUID. */
	if (!have_cpuid_p())
		identify_cpu_without_cpuid(c);

	/* x86_capability 的注释常出现 pattern: "Intel-defined", "AMD-defined",
	 * what does that mean? Guess: it denotes certain CPUID feature bit is firstly
	 * defined by Intel or AMD.
	 * Some feature bits stem from whole register value of CPUID output; while
	 * others scatter in CPUID leaves, put scattered bits into one word, which is
	 * called "Linux-defined bit".
	 *
	 * Other facts: different model of same vendor might also be different in their
	 * CPUID implementation, refer to mtrr_bp_init(); different vendor might have different
	 * CPUID output for the same leaf, such as CPUID.6.ECX.
	 *
	 * Full feature bit definition at arch/x86/include/asm/cpufeatures.h
	 */

	/* cyrix could have cpuid enabled via c_identify()*/
	if (have_cpuid_p()) {
		/* CPUID.00H & CPUID.01H */
		cpu_detect(c);

		/* 将 CPUID.00H 拿到的 Vendor Identification String，与代码定义的 struct
		 * cpu_dev.c_ident 比对(某 cpu 竟然提供 2 个 id string)，so, kernel knows
		 * which vendor's CPU it is running on. 保存在变量 this_cpu. */
		get_cpu_vendor(c);

		/* Get CPU capability, i.e., CPU feature, into boot_cpu_data.x86_capability[].
		 * CPU feature checking is often seen later via boot_cpu_has(bit) or cpu_has(bit).
		 *
		 * There are currently 19(NCAPINTS) feature words, 13 of which are common for
		 * all vendors, 2 of which are vendor-specific: CPUID_8086_0001_EDX of transmeta
		 * & CPUID_C000_0001_EDX of VIA/Cyrix/Centaur, 4 of which are called Linux-defined. 
		 *
		 * Notes about this function:
		 *   - 使用 sub-leaf via cpuid_count();
		 *   - 值得分析(在下文)的 cpu_has();
		 *   - 根据 feature bit 设置其他 boot_cpu_data 中相应 field.
		 *   - 可 force set/clear certain bit via cpu_caps_set[] & cpu_caps_cleared[].
		 * 	   下面很快看到 force set/clear 的例子。
		 */
		get_cpu_cap(c);
		/* Get real address bit width which are initialized before. */
		get_cpu_address_sizes(c);
		setup_force_cpu_cap(X86_FEATURE_CPUID);
		/* command line may also set/clear feature bit. */
		cpu_parse_early_param();

		/* Fine-tune x86_capability[]. 小众 x86 vendor 可以在此初始化 its
		 * specific feature word. */
		if (this_cpu->c_early_init)
			this_cpu->c_early_init(c);

		c->cpu_index = 0;
		/* Still fine-tune x86_capability[]. Refer to comments of
		 * struct cpuid_dependent_feature */
		filter_cpuid_features(c, false);

		/* 看起来是根据 CPU feature bit 设置一些变量。*/
		if (this_cpu->c_bsp_init)
			this_cpu->c_bsp_init(c);
	} else {
		setup_clear_cpu_cap(X86_FEATURE_CPUID);
	}

	setup_force_cpu_cap(X86_FEATURE_ALWAYS);

	/* Background: CPU bug, like "meltdown", "spectre", etc.
	 * Rough analysis：内置白名单 cpu_vuln_whitelist, 通过比对 CPU vendor, family,
	 * model, 确认 current CPU 是否在 whitelist.  According to code, it seems that
	 * 所有 CPU bug 本质都是 SPECULATION, 所以，如果 whitelist 定义某 CPU model 是
	 * NO_SPECULATION, 则无需 set any bug bit; SPECTRE 看起来在所有 CPU 都存在；
	 * 其他 CPU bug 还需 MSR_IA32_ARCH_CAPABILITIES 的值配合确认。*/
	cpu_set_bug_bits(c);
	...
}
```

**cpu_has** is macro defined as:

```c
#define cpu_has(c, bit)							\
	(__builtin_constant_p(bit) && REQUIRED_MASK_BIT_SET(bit) ? 1 :	\
	 test_cpu_cap(c, bit))
```

Two pieces of knowledge of **cpu_has** worth to be aware:

  1. GCC builtin function **__builtin_constant_p**: determine if a value is
known to be constant at compile time. Refer to it in [GCC manual](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#Other-Builtins).
  2. REQUIRED_MASK_BIT_SET involves tricky macro: BUILD_BUG_ON_ZERO, refer to [here](   https://stackoverflow.com/questions/9229601/what-is-in-c-code).

(For me)One keyword behind the trick: **anonymous bit-field**, it means: An unnamed bit-field structure member is useful for padding to conform to externally imposed layouts.

Another piece of info about REQUIRED_MASK_BIT_SET worth to know, check the header comments of *arch/x86/include/asm/required-features.h*. Linux kernel has minimum requirements for underlying CPU, these minimum requirements is expressed via feature bit in REQUIRED_MASKn, 当检查 CPU 是否支持某 feature bit 时，先从 REQUIRED_MASKn 确认是否属于 minimum requirements feature bit, 若不是，则由函数 test_cpu_cap 从 struct cpuinfo_x86.x86_capability[] 检查。

REQUIRED_MASK_BIT_SET 中重复了 BUILD_BUG_ON_ZERO(NCAPINTS != 19)，与社区交流
后理解了原因，且促成 commit cbb1133b563a63. The [original version](https://lkml.org/lkml/2019/8/28/60) of the commit maybe more friendly to newbie chinese english speaker. Simply: header file **cpufeature.h** & **required-features.h** 都使用了 **cpufeatures.h** 中的 CPU feature bit macro, 当 CPU feature 定义更新时，所有使用处都也相应更新，目前代码的做法提供了编译时检查。Tip: header file 被 include 到 .c 文件中才会被编译到，the including relationship is: **cpufeature.h**[1] include **processor.h**[2] include **cpufeatures.h**[3] include **required-features.h**[4], 所以 [1] 可以直接使用 [4] 中定义的 macro.

## early_ioremap_init

The function is going to involve the concept of "fixmap" in Linux kernel. Kernel doesn't have document for it, fixmap.h header comments said:

>fixmap.h: compile-time virtual memory allocation

理解了 fixmap, early_ioremap_init 的内容迎刃而解。**fixmap**, as the name tells，part of virtual address range have **fixed** assignment, the assignment is hard coded.

Linux kernel statically(compile-time) allocates certain virtual address range for specific usage, the range is initially of FIXMAP_PMD_NUM(2) PMD entries, i.e., 4M, specifically, 506th & 507th(FIXMAP_PMD_TOP) entry in level2_fixmap_pgt, it can be confirmed by code of early page table construction in head_64.S. fixmap range is [FIXADDR_START, FIXADDR_TOP + PAGE_SIZE), which is divided by PAGE_SIZE, and indexed by *enum fixed_addresses*. The trick is, pages of fixmap range is inversely(**逆向**) indexed, i.e., 0 indexes address FIXADDR_TOP, max number indexes FIXADDR_START.

fixmap range 中还有细分：部分 range 用于明确的映射，部分 range 用于 boot time 的各种临时 映射, 比如访问 firmware data. Refer to *__end_of_permanent_fixed_addresses* & *__end_of_fixed_addresses*.

先分析上面提到的几个 macro:

```c
/* Refer to VSYSCALL_ADDR first.
 * PAGE_SIZE = 4k, PMD_SHIFT = 21. 简单心算可知 FIXADDR_TOP = VSYSCALL_ADDR + 2M - 4k,
 * 是地址 -8M 减去 4k 的位置，是 level2_fixmap_pgt 中 507th entry(2M) 中最后一个 PTE
 * 的地址，目的是使 0 索引 FIXADDR_TOP.
 */
#define FIXADDR_TOP	(round_up(VSYSCALL_ADDR + PAGE_SIZE, 1<<PMD_SHIFT) - \
			 PAGE_SIZE)

/* Documentation/x86/x86_64/mm.rst: 起始于 0xffffffffff600000 的 4k 用于
 * legacy vsyscall ABI. (-10UL << 20) = ffffffffff600000, 即 -10M, 已对齐到 2M.
 * 虚拟地址 -10M 恰好对应 level2_fixmap_pgt 中 507th entry.
 */
#define VSYSCALL_ADDR (-10UL << 20)
```

Index number is defined by **enum fixed_addresses**, 不同情况下其 index value 定义不同，i.e., the required fix-mapped space size differs, so FIXADDR_START's value also differs:

```c
#define FIXADDR_SIZE		(__end_of_permanent_fixed_addresses << PAGE_SHIFT)
#define FIXADDR_START		(FIXADDR_TOP - FIXADDR_SIZE)

/* By intuition, the value of FIXADDR_START is problematic, should it be plused by 4k? */
```

*__end_of_permanent_fixed_addresses* is defined in **enum fixed_addresses**, which has many #ifdef included, hard to read, so, 适当美化方便阅读理解:

```c
/* CONFIG_X86_VSYSCALL_EMULATION depends on X86_64, defaults to Y, suggests that
 * vsyscall exists in fixmap range. (不理解 vsyscall 不影响对本段代码的理解。)
 * 计算可知, VSYSCALL_PAGE = 511. 与 X86_64 不同，X86_32 index value starts with 0.
 *
 * (看完下面其他代码分析后，再来看下面这段的总结)
 * 可以看出，w/ CONFIG_X86_VSYSCALL_EMULATION 时，2 个 PMD entry(2M) 中，后者(高地址)
 * 用于 vsyscall 映射，前者(低地址)用于 FIX_DBGP_BASE 及之后的映射；
 * w/o CONFIG_X86_VSYSCALL_EMULATION 时，后者(高地址)直接作其他用途。
 */
enum fixed_addresses {
	...
#ifdef CONFIG_X86_VSYSCALL_EMULATION
	VSYSCALL_PAGE = (FIXADDR_TOP - VSYSCALL_ADDR) >> PAGE_SHIFT,
#endif

	/* CONFIG_X86_VSYSCALL_EMULATION 默认 y, 所以 FIX_DBGP_BASE = 512,
	 * 指向 last PTE of page table pointed by 506th PMD entry(2M). */
	FIX_DBGP_BASE,
	FIX_EARLYCON_MEM_BASE,
	...
	/* 看到 APIC, 开始悟到一些： APIC 有很多寄存器 */
#ifdef CONFIG_X86_LOCAL_APIC
	FIX_APIC_BASE,	/* local (CPU) APIC) -- required for SMP or not */
#endif

#ifdef CONFIG_X86_IO_APIC
	FIX_IO_APIC_BASE_0,
	FIX_IO_APIC_BASE_END = FIX_IO_APIC_BASE_0 + MAX_IO_APICS - 1,
#endif

	...

#ifdef CONFIG_ACPI_APEI_GHES
	/* Used for GHES mapping from assorted contexts */
	FIX_APEI_GHES_IRQ,
	FIX_APEI_GHES_NMI,
#endif

	__end_of_permanent_fixed_addresses,

	/*
	 * 512 temporary boot-time mappings, used by early_ioremap(),
	 * before ioremap() is functional.
	 *
	 * If necessary we round it up to the next 512 pages boundary so
	 * that we can have a single pmd entry and a single pte table:
	 */
	/* Kernel 初始化过程中，常需要临时 map 访问某些物理地址，访问结束便 unmap.
	 * Will see a lot of it later.
	 *
	 * For intelligibility, simplify the following conditional expression to:
	 *
	 *    x ^ (x + 0x1FF)
	 *	  &
	 *	  0xfffffc00(PTRS_PER_PTE=1024) 或 0xfffffe00(PTRS_PER_PTE=512)
	 *
	 * (PTRS_PER_PTE = 1024 only in CONFIG_X86_32 w/o PAE, 其他情况下都 = 512.)
	 *
	 * When PTRS_PER_PTE = 1024(-PTRS_PER_PTE = 0xfffffc00), PMD entry point to
	 * 1024 PTEs, covers 4M.
	 * Condition Expression result = TRUE if x > 512; FALSE if x <= 512.
	 * If TRUE(x > 512):   FIX_BTMAP_END 将向低地址方向对齐到 TOTAL_FIX_BTMAPS(512),
	 *                     即，FIX_BTMAP_END index next(逆向) PMD entry(4M) 中最末
	 *                     page, or index next(逆向) PMD entry(4M) 中 512th entry, etc.
	 * If FALSE(x <= 512): FIX_BTMAP_END = x. 说明本 PMD entry(4M) 中有 > 512 个 PTE 空闲.
	 *
	 * PTRS_PER_PTE = 512, -PTRS_PER_PTE = 0xfffffe00, 即 X86_64 or X86_32 w/ PAE,
	 * PMD entry has 512 entries, covers 2M 时:
	 * Condition Expression result *always* = TRUE.
	 * FIX_BTMAP_END 向低地址方向对齐到 TOTAL_FIX_BTMAPS, 即 FIX_BTMAP_END index
	 * next(逆向) PMD entry(2M) 中的 last page.
	 *
	 * "round it up to next 512 pages boundary if necessary" finally make
	 * sense: if index x > 511 */
#define NR_FIX_BTMAPS		64 /* 看起来像 long 的 bit 数 */
#define FIX_BTMAPS_SLOTS	8  /* 需要 512 个 4k mapping */
#define TOTAL_FIX_BTMAPS	(NR_FIX_BTMAPS * FIX_BTMAPS_SLOTS)
	FIX_BTMAP_END =
	 (__end_of_permanent_fixed_addresses ^
	  (__end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS - 1)) &
	 -PTRS_PER_PTE
	 ? __end_of_permanent_fixed_addresses + TOTAL_FIX_BTMAPS -
	   (__end_of_permanent_fixed_addresses & (TOTAL_FIX_BTMAPS - 1))
	 : __end_of_permanent_fixed_addresses,

	/* 了解了 fixmap 空间是逆向索引，就明白为何 enum 中 FIX_BTMAP_END 定义在前，而
	 * FIX_BTMAP_BEGIN 在后面，END, BEGIN 是虚拟地址增长的角度。
	 *
	 * [Tip] 由上面表达式的分析可知，有 3 种情况:
	 *
	 *   1. (PTRS_PER_PTE = 1024 && FIX_BTMAP_END <= 512), then FIX_BTMAP_BEGIN
	 *      < 1023, so FIX_BTMAP_END & FIX_BTMAP_BEGIN cover all the pages in the
	 *      same PMD entry(4M).
	 *
	 *   2. (PTRS_PER_PTE = 1024 && FIX_BTMAP_END > 512), FIX_BTMAP_END 对齐 512,
	 *      意味着它 index next(逆向) PMD entry(4M) 中的最末 PTE, or 512th entry,
	 *      i.e., FIX_BTMAP_END & FIX_BTMAP_BEGIN always index PTEs that belonging
	 *      to the same PMD entry.
	 *
	 *   3. (PTRS_PER_PTE = 512), FIX_BTMAP_END 总是 512(TOTAL_FIX_BTMAPS) aligned,
	 *      即总是 index next(逆向) PMD entry(2M) 中 last PTE, 所以 FIX_BTMAP_BEGIN
	 *      也总是 index next(逆向) PMD entry(2M) 中 1st PTE, i.e., 他们始终
	 *      index 同一个 PMD entry.
	 *
	 * FIX_BTMAP_BEGIN & FIX_BTMAP_END 必须 index 同一个 PMD entry, [Tip] 显示 3 种情况
	 * 都满足此条件。此外，函数 early_ioremap_init() 中也有 runtime check, enforce 此条件。
	 */
	FIX_BTMAP_BEGIN = FIX_BTMAP_END + TOTAL_FIX_BTMAPS - 1,

	...

	__end_of_fixed_addresses
};
```

上面是 fixmap 的数据结构设计，the basic usage is conversion between index & virtual address:

```c
/*
 * 'index to address' translation. If anyone tries to use the idx
 * directly without translation, we catch the bug with a NULL-deference
 * kernel oops. Illegal ranges of incoming indices are caught too.
 */
static __always_inline unsigned long fix_to_virt(const unsigned int idx)
{
	/* index 不能超过定义的 max value. */
	BUILD_BUG_ON(idx >= __end_of_fixed_addresses);
	return __fix_to_virt(idx);
}

static inline unsigned long virt_to_fix(const unsigned long vaddr)
{
	BUG_ON(vaddr >= FIXADDR_TOP || vaddr < FIXADDR_START);
	return __virt_to_fix(vaddr);
}

/* 逆向映射, index 0 => FIXADDR_TOP */
#define __fix_to_virt(x)	(FIXADDR_TOP - ((x) << PAGE_SHIFT))
#define __virt_to_fix(x)	((FIXADDR_TOP - ((x)&PAGE_MASK)) >> PAGE_SHIFT)
```

Then come to initialization of fixmap, i.e., early_ioremap_init:

```c
static pte_t bm_pte[PAGE_SIZE/sizeof(pte_t)] __page_aligned_bss;
...

/* Fixed address assignment is done before compilation, so, runtime check to assure
 * they behaves. */
void __init early_ioremap_init(void)
{
	pmd_t *pmd;

	/* According to analysis of "enum fixed_addresses", address (index 0 + PAGE_SIZE)
	 * is aligned to 2M. */
#ifdef CONFIG_X86_64
	BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
#else
	...
#endif

	/* The 512 pages from FIX_BTMAP_BEGIN to FIX_BTMAP_END are divided into 8 slots
	 * per 64 pages. Slot is also indexed normally, i.e., low to high address.
	 * Store slot address into slot_virt[].
	 */
	early_ioremap_setup();

	/* 找到 FIX_BTMAP_BEGIN 对应的 PMD entry. */
	pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
	/* bm_pte 表示 page table, 刚找到的 PMD entry *pmd* 将指向它 */
	memset(bm_pte, 0, sizeof(bm_pte));
	pmd_populate_kernel(&init_mm, pmd, bm_pte);

	/* Tips: head_64.S 中构建 early page table 时，已为 fixmap 空间初始化了 506th,
	 * 507th 两个 PMD entry. 正常情况(w/ CONFIG_X86_VSYSCALL_EMULATION)下： 507th
	 * entry 用于 vsyscall emulation mapping; 506th entry 用于 [FIX_DBGP_BASE，
	 * FIX_BTMAP_END) 的 mapping.
	 * 而地址 [FIX_BTMAP_END - FIX_BTMAP_BEGIN] 未在 head_64.s 中初始化，显然，it
	 * belongs to 505th PMD entry, 上述代码就是相应的页表初始化。*/

	/*
	 * The boot-ioremap range spans multiple pmds, for which
	 * we are not prepared:
	 */
	/*
	 * Macro __FIXADDR_TOP is only used under CONFIG_X86_32, because macro
	 * __fix_to_virt needs to access its value. Under x86_32, variable __FIXADDR_TOP
	 * is defined in arch/x86/mm/pgtable_32.c, during compilation, its value can't be
	 * determined until linking, so the value of expression __fix_to_virt() also can't
	 * be evaluated.
	 *
	 * attribute __error__() 的使用，需要在编译时知道 if(__FIXADDR_TOP) 中的值，
	 * 所以定义一个同名的宏，临时使用。这暗示了一个 tip: 同名的 variable 和 macro,
	 * 编译时优先使用 macro. 因为 macro 处理发生在 pre-processing,而 variable 处理
	 * 发生在 compilation.   下方 demo 程序可用于验证上述分析。
	 *
	 * Make sure FIX_BTMAP_BEGIN & FIX_BTMAP_END index the pages which exist in
	 * the same PMD.  详细分析在 FIX_BTMAP_BEGIN 的定义处，共 3 种情况。
	 */
#define __FIXADDR_TOP (-PAGE_SIZE)
	BUILD_BUG_ON((__fix_to_virt(FIX_BTMAP_BEGIN) >> PMD_SHIFT)
		     != (__fix_to_virt(FIX_BTMAP_END) >> PMD_SHIFT));
#undef __FIXADDR_TOP

	/* 如果 FIX_BTMAP_BEGIN 和 FIX_BTMAP_END 不在同一个 PMD 中，就一堆 printk.
	 * 上面已有 BUILD_BUG_ON 作 enforcement, 这里需要吗? 还以为可以发 patch 删掉，研究
	 * 了一大圈，发现并不是完全没有意义，因为 BUILD_BUG_ON 仅是使用 GCC 的 attribute
	 * __error__, 而现在 kernel 还要支持 clang, clang 中并没有这个功能，即它在 clang
	 * 下没用，这时如果发生了这个错误，下面的 printk 可以提供一些有用信息 */
	if (pmd != early_ioremap_pmd(fix_to_virt(FIX_BTMAP_END))) {
		WARN_ON(1);
		/* 一堆 printk，略过 */
	}
}
```

用于验证上述 X86_32 下 __FIXADDR_TOP 分析的 demo program(仅编译: `gcc -c xx.c -o x.o`):
```
extern unsigned long __FIXADDR_TOP;
#define FIXADDR_TOP     ((unsigned long)__FIXADDR_TOP)
//int __FIXADDR_TOP = 1;

extern void nocompiling(void) __attribute__((__error__("NO compile on purpose")));

int main()
{
//#define __FIXADDR_TOP 0
    if (FIXADDR_TOP)
        nocompiling();
//#undef __FIXADDR_TOP

    return 0;
}
```
(注意：自己调整代码细节进行验证。)

Comments of FIX_BTMAP_END say: *512 temporary boot-time mappings, used by early_ioremap()*, but actually , temporary mapping is also used in following several situation:

```c
/* After searching its caller, it can be deduced that: register or ROM of certain
 * device on south bridge of PC would be mapped into CPU's physical address space,
 * like Apple airport, PCI serial port, etc.
 */
/* Remap an IO device */
void __init __iomem *
early_ioremap(resource_size_t phys_addr, unsigned long size)
{
	return __early_ioremap(phys_addr, size, FIXMAP_PAGE_IO);
}

/* During kernel initialization, many data(all from firmware?) in memory
 * need "one-time" access, like ACPI table.
 */
/* Remap memory */
void __init *
early_memremap(resource_size_t phys_addr, unsigned long size)
{
	pgprot_t prot = early_memremap_pgprot_adjust(phys_addr, size,
						     FIXMAP_PAGE_NORMAL);

	return (__force void *)__early_ioremap(phys_addr, size, prot);
}

#ifdef FIXMAP_PAGE_RO
/* 同上，区别仅是 Read Only or not.*/
void __init *
early_memremap_ro(resource_size_t phys_addr, unsigned long size)
{
	pgprot_t prot = early_memremap_pgprot_adjust(phys_addr, size,
						     FIXMAP_PAGE_RO);

	return (__force void *)__early_ioremap(phys_addr, size, prot);
}
#endif

#ifdef CONFIG_ARCH_USE_MEMREMAP_PROT
/* user-defined pgprot_t value */
void __init *
early_memremap_prot(resource_size_t phys_addr, unsigned long size,
		    unsigned long prot_val)
{
	return (__force void *)__early_ioremap(phys_addr, size,
					       __pgprot(prot_val));
}
#endif

```

The first 3 callers of __early_ioremap have different pgprot_t value, essentially all are __PAGE_KERNEL, difference is whether has flag: _ENC, __RW or not. early_memremap_pgprot_adjust 仅判断 page attribute 是否需要 encryption flag, 并相应调整。FIXMAP_PAGE_IO has no _ENC flag, because encryption is targeted for memory, not I/O device.

For x86, if !CONFIG_MMU, i.e., paging is not enabled, virtual address = physical address. Otherwise, page mapping it:

```c
#ifdef CONFIG_MMU
...

/* From user perspective, fixmap area is used by slot(64 pages), but page mapping
 * is done by page. So temporary mapping area has most FIX_BTMAPS_SLOTS users
 * at the same time.
 */
static void __init __iomem *
__early_ioremap(resource_size_t phys_addr, unsigned long size, pgprot_t prot)
{
	unsigned long offset;
	resource_size_t last_addr;
	unsigned int nrpages;
	enum fixed_addresses idx;
	int i, slot;

	WARN_ON(system_state >= SYSTEM_RUNNING);

	/* 寻找一个未被占用的 slot. */
	slot = -1;
	for (i = 0; i < FIX_BTMAPS_SLOTS; i++) {
		if (!prev_map[i]) {
			slot = i;
			break;
		}
	}

	if (WARN(slot < 0, "%s(%pa, %08lx) not found slot\n",
		 __func__, &phys_addr, size))
		return NULL;

	/* Don't allow wraparound or zero size */
	last_addr = phys_addr + size - 1;
	if (WARN_ON(!size || last_addr < phys_addr))
		return NULL;

	prev_size[slot] = size;
	/*
	 * Mappings have to be page-aligned
	 */
	offset = offset_in_page(phys_addr);
	phys_addr &= PAGE_MASK;
	size = PAGE_ALIGN(last_addr + 1) - phys_addr;

	/*
	 * Mappings have to fit in the FIX_BTMAP area.
	 */
	/* mapping size 不可大于 slot(64 pages). */
	nrpages = size >> PAGE_SHIFT;
	if (WARN_ON(nrpages > NR_FIX_BTMAPS))
		return NULL;

	/*
	 * Ok, go for it..
	 */
	/* Mapping from the 1st page of the slot. */
	idx = FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*slot;
	while (nrpages > 0) {
		/* According to the usage of after_paging_init, it can be told that it is only
		 * useful under x86_32, which means x86_64 gose to the "else" branch. */
		if (after_paging_init)
			__late_set_fixmap(idx, phys_addr, prot);
		else
			__early_set_fixmap(idx, phys_addr, prot);

		phys_addr += PAGE_SIZE;
		--idx;
		--nrpages;
	}
	WARN(early_ioremap_debug, "%s(%pa, %08lx) [%d] => %08lx + %08lx\n",
	     __func__, &phys_addr, size, slot, offset, slot_virt[slot]);

	/* 不言而喻: 因为是从 slot 中第一个 page 开始 mapping. */
	prev_map[slot] = (void __iomem *)(offset + slot_virt[slot]);
	return prev_map[slot];
}

#else
...
#endif
```

## e820__memory_setup
```
/*
 * Calls e820__memory_setup_default() in essence to pick up the firmware/bootloader
 * E820 map - with an optional platform quirk available for virtual platforms
 * to override this method of boot environment processing:
 */
/* Specific platform may differ in processing E820 data from firmware. */
void __init e820__memory_setup(void)
{
	char *who;

	/* This is a firmware interface ABI - make sure we don't break it: */
	BUILD_BUG_ON(sizeof(struct boot_e820_entry) != 20);

	/* Copy boot_params.e820_table of firmware to e820_table(the main E820 table
	 * of kernel), and sanitize the data by e820__update_table(), like sorting,
	 * overlap merging.
	 */
	who = x86_init.resources.memory_setup();

	/* Then copy it to _kexec and _firmware. Check the comments of these 3 e820 table
	 * variables for the differences. */
	memcpy(e820_table_kexec, e820_table, sizeof(*e820_table_kexec));
	memcpy(e820_table_firmware, e820_table, sizeof(*e820_table_firmware));

	pr_info("BIOS-provided physical RAM map:\n");
	e820__print_table(who);
}
```

Under 32-bit or 64-bit boot protocol, boot loader is responsible for filling out this piece of data，comments of e820__memory_setup_default() can confirm the conclusion. The sanitization of E820 data employs a complex algorithm in e820__update_table().

In the midst of E820 processing call-chain, encounter the branch predication statement:likely(), for the 1st time. In my kernel configuration, it is defined as:

```c
# define likely(x)	__builtin_expect(!!(x), 1)
# define unlikely(x)	__builtin_expect(!!(x), 0)
```

which is [GCC built-in function](Refer: https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html).

## parse_early_param -> parse_early_options

parse_early_param() has local static variable to prevent it from calling again.

```c
void __init parse_early_options(char *cmdline)
{
	/* Tip: Kernel parameter parsing happens at different phases, early parsing
	 * is the first of them. Parameter parsing finally all go to parse_args().
	 *
	 * 入参 struct kernel_param *params 为 NULL, 使得 early parsing 直接落入
	 * do_early_param().
	 */
	parse_args("early options", cmdline, NULL, 0, 0, 0, NULL,
		   do_early_param);
}

/* Check for early params. */
/* Early parsing only parses parameters against the ones defined by early_param(),
 * i.e., the ones set the "early" flag.
 * "console" is actually implemented as "earlycon" in code, which is provided by
 * drivers/tty/serial/earlycon.c
 */
static int __init do_early_param(char *param, char *val,
				 const char *unused, void *arg)
{
	const struct obs_kernel_param *p;

	for (p = __setup_start; p < __setup_end; p++) {
		if ((p->early && parameq(param, p->str)) ||
		    (strcmp(param, "console") == 0 &&
		     strcmp(p->str, "earlycon") == 0)
		) {
			if (p->setup_func(val) != 0)
				pr_warn("Malformed early option '%s'\n", param);
		}
	}
	/* We accept everything at this stage. */
	return 0;
}
```

**__setup_start** & **__setup_end** are used to mark .init.setup section's boundary, which is defined in linker script as:

```c
/*
 * Only for really core code.  See moduleparam.h for the normal way.
 *
 * Force the alignment so the compiler doesn't space elements of the
 * obs_kernel_param "array" too far apart in .init.setup.
 */
#define __setup_param(str, unique_id, fn, early)			\
	static const char __setup_str_##unique_id[] __initconst		\
		__aligned(1) = str; 					\
	static struct obs_kernel_param __setup_##unique_id		\
		__used __section(.init.setup)				\
		__attribute__((aligned((sizeof(long)))))		\
		= { __setup_str_##unique_id, fn, early }

/* 常见的定义 parameter 的方式 */
#define __setup(str, fn)						\
	__setup_param(str, fn, fn, 0)

/*
 * NOTE: fn is as per module_param, not __setup!
 * Emits warning if fn returns non-zero.
 */
#define early_param(str, fn)						\
	__setup_param(str, fn, fn, 1)

/* 目前(2020/11)仅有一处(gbpages/nogbpages)使用。初看有些困惑，若 kconfig 配置不支持
 * feature, 依然可以通过 parameter 强行 enable, 是什么逻辑? 也许就是想提供这个功能。
 * bfb33bad83f650f has info about it. */
#define early_param_on_off(str_on, str_off, var, config)  \
 /* Omit the definition. */
```

Comments "*Only for really core code. See moduleparam.h for the normal way*" reveals interesting info, refer to Documentation/admin-guide/kernel-parameters.rst for introduction of kernel parameter. Simply, kernel has 2 implementations for parameter handling, one is **__setup_param**; the other is defined in moduleparam.h.

A few tips for **__setup_param**:

  1. It seems to be the original parameter mechanism, used by different wrappers, its simplicity can be told by definition of *struct obs_kernel_param*.
  2. Not used by modules.
  3. *struct obs_kernel_param* is **obsolete**.
  4. Two parameter mechanism use different data structure, and in turn different sections for storing it.

### console VS earlycon VS earlyprintk

do_early_param() shows a little bit information about **console** & **earlycon**,  and there is still another relative parameter **earlyprintk**, what's the relation & difference among them? Reference: [Early console confusion
](https://lists.archive.carbon60.com/linux/kernel/1279156).

Strictly, string **console** is not a kernel parameter as the other two, which are defined through *early_param()*. There are only 2 places which parse it:

```
arch/x86/boot/early_serial_console.c:  if (cmdline_find_option("console", optstr, sizeof(optstr)) <= 0)
init/main.c:		    (strcmp(param, "console") == 0 &&
```

one is in x86 setup image, which only take the option of *uart[8250],io,<addr>[,options]*; the other is in VO. As said above, in VO, **console** is actually implemented via **earlycon**.

**earlycon** is implemented in:
```
drivers/tty/serial/earlycon.c:early_param("earlycon", param_setup_earlycon);
```
**earlyprintk** is implemented by arch itself:
```
arch/m68k/kernel/early_printk.c:early_param("earlyprintk", setup_early_printk);
arch/s390/kernel/early_printk.c:early_param("earlyprintk", setup_early_printk);
arch/arm/kernel/early_printk.c:early_param("earlyprintk", setup_early_printk);
arch/sh/kernel/sh_bios.c:early_param("earlyprintk", setup_early_printk);
arch/um/kernel/early_printk.c:early_param("earlyprintk", setup_early_printk);
arch/x86/kernel/early_printk.c:early_param("earlyprintk", setup_early_printk);
```

Both finally call **register_console(struct console *newcon)** for logging service. The difference seems to be: **earlyprintk** has limited built-in consoles, which are well-defined concepts in industry, such as serial port, VGA, refer to **setup_early_printk()**. While **earlycon** can support lots of devices, which are defined via **OF_EARLYCON_DECLARE** and locates in drivers/tty/serial/.

After several days of skimming through the code of register_console()** & **printk(), got the impression: printk() records all logs into its ring buffer **static char __log_buf[__LOG_BUF_LEN]** via **log_store()**, the log buffer is configurable via **CONFIG_LOG_BUF_SHIFT**. Log buffer is managed through **struct printk_log**. New log will also be selectively outputed to console via **console_unlock()** --> **call_console_drivers()**, by applying log level against **console_loglevel**.

In a word, printk() 的 log 都记录在 kernel, 可通过 `dmesg` 检阅; 根据 log level 选择性输出到 console.

## init_hypervisor_platform

```
static const __initconst struct hypervisor_x86 * const hypervisors[] =
{
#ifdef CONFIG_XEN_PV
	&x86_hyper_xen_pv,
#endif
#ifdef CONFIG_XEN_PVHVM
	&x86_hyper_xen_hvm,
#endif
	&x86_hyper_vmware,
	&x86_hyper_ms_hyperv,
#ifdef CONFIG_KVM_GUEST
	&x86_hyper_kvm,
#endif
#ifdef CONFIG_JAILHOUSE_GUEST
	&x86_hyper_jailhouse,
#endif
#ifdef CONFIG_ACRN_GUEST
	&x86_hyper_acrn,
#endif
};

 void __init init_hypervisor_platform(void)
{
	const struct hypervisor_x86 *h;

	/* 进入此函数可观察到，x86 支持的 hypervisor 平台列表定义在 hypervisors[]. 每个
	 * hypervisor 都有自己的 detect 函数。TBD: 分析 KVM's & XEN's. 未来分析虚拟化
	 * 时在详细研究下面的代码。 */
	h = detect_hypervisor_vendor();

	if (!h)
		return;

	copy_array(&h->init, &x86_init.hyper, sizeof(h->init));
	copy_array(&h->runtime, &x86_platform.hyper, sizeof(h->runtime));

	x86_hyper_type = h->type;
	x86_init.hyper.init_platform();
}
```

## probe_roms

Some devices like video card also has its firmware. So, the "BIOS" people often refers to is also called system BIOS, while video firmware is called [Video BIOS](https://en.wikipedia.org/wiki/Video_BIOS). Info of [memory map of video BIOS](https://wiki.osdev.org/Memory_Map_(x86)#ROM_Area) is critical for Understanding the code.

Didn't authoritative reference for firmware image format, but there is a piece of useful info for it from [《PCI System Architecture》](http://www.informit.com/store/pci-system-architecture-9780201309744), chapter 20: Expansion ROMS.

Code image has 4 components: ROM header, ROM data structure, Run-time code, Initialization code. 此处代码 focus on ROM header.

ROM image 开头是 ROM header, first 2 bytes is called ROM Signature: must contain 0xAA55, identifying this as a device ROM. This has always been the signature used for a device ROM in any PC-compatible machine.

![rom signature](romsig.png)

ROM signature 后，the 3rd byte 表示: overall size of the image(in 512 byte increments).

![rom signature](afteromsig.png)

[Wikipedia](https://en.wikipedia.org/wiki/BIOS#Initialization) has a general description for the whole process.

With background knowledge above, Let's get back to code:

```c
void __init probe_roms(void)
{
	const unsigned char *rom;
	unsigned long start, length, upper;
	unsigned char c;
	int i;

	/* video rom */
	/* 根据上面 rom's memory map 可知: 0xC0000 是 video BIOS 的起始地址，0xC8000 用于
	 * 其他设备 rom(firmware), 这也许就是 adapter 的含义。一共 8k 的 range, 以 2k 的
	 * step size 检查 video BIOS, 虽然不明白为什么是 2k step size.*/
	upper = adapter_rom_resources[0].start;
	for (start = video_rom_resource.start; start < upper; start += 2048) {
		/* 只是对 __va() 的简单封装，上文有分析过，涉及 page fault 中断服务。
		 * 拿到 video BIOS 所在物理地址对应的虚拟地址，才可以在代码中访问它 */
		rom = isa_bus_to_virt(start);
		/* 读取 first 2 bytes of ROM. 调用了实现复杂的 probe_kernel_address, TBD.*/
		if (!romsignature(rom))
			continue;

		video_rom_resource.start = start;

		if (probe_kernel_address(rom + 2, c) != 0)
			continue;

		/* 0 < length <= 0x7f * 512, historically */
		length = c * 512;

		/* if checksum okay, trust length byte */
		/* ROM checksum 机制忘记在哪儿看到，很简单：ROM 中所有 byte value 相加等于 0，
		 * 说明 ROM 是 OK 的。*/
		if (length && romchecksum(rom, length))
			video_rom_resource.end = start + length - 1;

		/* 这才是真正的重点, TBD. */
		request_resource(&iomem_resource, &video_rom_resource);
		break;
	}

	/* 处理完 video rom, 将其结束地址对齐到 2k boundary. */
	start = (video_rom_resource.end + 1 + 2047) & ~2047UL;
	if (start < upper)
		start = upper;

	/* system rom */
	/* 对其他的 ROM 也 request resource. system_rom_resource 是预定义的全局变量，描述
	 * [0xf0000 - 0xfffff], 即我们常说的 BIOS 的位置(也叫 Motherboard BIOS)。*/
	request_resource(&iomem_resource, &system_rom_resource);
	upper = system_rom_resource.start;

	/* 都是外设的 ROM，代码却做了区分，extension ROM VS adapter ROM, 暂不了解二者区别。*/

	/* check for extension rom (ignore length byte!) */
	rom = isa_bus_to_virt(extension_rom_resource.start);
	if (romsignature(rom)) {
		length = resource_size(&extension_rom_resource);
		if (romchecksum(rom, length)) {
			request_resource(&iomem_resource, &extension_rom_resource);
			upper = extension_rom_resource.start;
		}
	}

	/* check for adapter roms on 2k boundaries */
	for (i = 0; i < ARRAY_SIZE(adapter_rom_resources) && start < upper; start += 2048) {
		rom = isa_bus_to_virt(start);
		if (!romsignature(rom))
			continue;

		if (probe_kernel_address(rom + 2, c) != 0)
			continue;

		/* 0 < length <= 0x7f * 512, historically */
		length = c * 512;

		/* but accept any length that fits if checksum okay */
		if (!length || start + length > upper || !romchecksum(rom, length))
			continue;

		adapter_rom_resources[i].start = start;
		adapter_rom_resources[i].end = start + length - 1;
		request_resource(&iomem_resource, &adapter_rom_resources[i]);

		start = adapter_rom_resources[i++].end & ~2047UL;
	}
}

int request_resource(struct resource *root, struct resource *new)
{
	struct resource *conflict;

	conflict = request_resource_conflict(root, new);
	return conflict ? -EBUSY : 0;
}

/* 开始涉及读写锁的操作，值的研究，TBD. */
struct resource *request_resource_conflict(struct resource *root, struct resource *new)
{
	struct resource *conflict;

	write_lock(&resource_lock);
	conflict = __request_resource(root, new);
	write_unlock(&resource_lock);
	return conflict;
}
```
## mtrr_bp_init

Memory Type Range Registers(MTRR) is documented in Intel SDM 3a, chapter 11.11, it collaborate with Page Attribute Table(PAT) for managing memory type; PAT(documented at Intel SDM 3a, chapter 11.12) is a companion feature to the MTRR, in other words, PAT depends on MTRR, so in code, PAT is only initialized via mtrr_bp_pat_init() when MTRR enabled.

MTRR quick start: A mechanism for associating memory types with physical address ranges in system memory, allow processor optimize operations for different memory such as RAM, ROM, frame-buffer memory, memory-mapped I/O devices. Up to 96 memory ranges can be defined, 88 among them are fixed ranges which are for the 1st Megabytes, the rest are for variable ranges. Following a hardware reset, all the fixed and variable MTRRs are disabled, make all physical memory uncacheable. Typically, firmware, BIOS for example, will configure the MTRRs. MTRR supports 5 memory types: Uncacheable(UC), Write Combining(WC), Write-through(WT), Write-protected(WP), Writeback(WB). Region's base address & size in MTRR must be 4K aligned.   A [reference](https://www.kernel.org/doc/html/latest/x86/mtrr.html) for MTRR usage in Linux kernel.

PAT quick start: PAT is a companion feature to the MTRRs, MTRR maps memory type to regions of physical address space, while PAT maps memory types to pages within linear address space. PAT extends the function of PCD & PWT bit of page table to allow 6(5 of MTRR, a new Uncached(UC-)) types to be assigned dynamically to pages of linear address. With PAT feature, PAT bit of page-table or page-directory entry works with PCD, PWT bit, to define the memory type for the page. NOTE: PAT bit exist only when entry maps to page frame. PAT involves 2 kinds of encoding: Memory type encoding for IA32_PAT MSR[1]; PAT+PCD+PWT encoding as the index to PAT entry of IA32_PAT MSR[2];

  1. Intel SDM 3a: Table 11-10. Memory Types That Can Be Encoded With PAT.
  2. Intel SDM 3a: Table 11-11. Selection of PAT Entries with PAT, PCD, and PWT Flags.

In case of any type conflicts(inside MTRR or between MTRR & PAT, etc): CPU choose the conservative memory type.

MSR belongs to CPU core, in SMP, each CPU has MTRR & PAT MSR, need to maintain register value consistency among all CPUs. 这是 mtrr_bp_pat_init 中存在貌似不相关的复杂操作的原因。Refer to: 11.11.8 MTRR Considerations in MP Systems, 11.12.4 Programming the PAT.

Variable Range MTRR 中 region range 的计算目前还没明白，暂且记下其 mask 计算原则：
> Address_Within_Range AND PhysMask = PhysBase AND PhysMask

For Intel, MTRR feature bit is defined in CPUID_1_EDX. But for other vendors, it may not be the case，for example, the code shows: certain 32-bit AMD CPU may support MTRR but not has a feautre bit(X86_FEATURE_K6_MTRR).

可能是因为 MTRR 寄存器众多，操作复杂，为 MTRR 操作专门定义了 struct mtrr_ops. Intel 使用 *generic_mtrr_ops*, 其他 vendor 有各自的 struct mtrr_ops. 从代码看，原来 AMD 只支持 2 个 variable range MTRR? 但 AMD SPEC 说支持 8 个.

get_mtrr_state() is to get info of MTRR MSR: Fixed-range, Variable-range, default type, enable or not, etc, into mtrr_state.

PAT is initialized via mtrr_bp_pat_init() --> pat_init(): Programming PAT MSR & initialize PAT table(__cachemode2pte_tbl[] & __pte2cachemode_tbl[]) with an OS-defined value. __init_cache_modes() is for constructing PAT table, encode the value of PAT+PCD+PWT needed by page table, store them in table, for future quick usage.  After Power-Up or reset, the default value of IA32_PAT MSR is not 0(refer to Intel SDM 3a, Table 11-12. Memory Type Setting of PAT Entries Following a Power-up or Reset), if IA32_PAT MSR = 0, implies PAT is disabled explicitly by firmware.

mtrr_cleanup() sanitize the value of Variable Range MTRRs, and write the purged data back to MTRRs. Convoluted operation, omit.

## mtrr_trim_uncached_memory

MTRR 的初始化(mtrr_bp_init) 后的数据在此函数中取出使用。函数 comments 的思考：本函数的存在是因为 Buggy BIOS 没有合理设置 MTRR?, What's the reasonable setup? 理论上，firmware 可以任意设置 MTRR，也就是说，整个 RAM 空间可以任意设置 memory type, 但从 comments 看，Linux kernel 要求自己使用的所有 memory 必须是 WB 类型，Why enforcement?

这篇关于 caching/MBRR/PAT 的[科普](https://lwn.net/Articles/282250/) 能够解答上述部分疑问： memory 相对于 CPU 很慢，所以 caching 很必要。不同类型的 memory 使用场景不同，需要合理的 memory type，或曰 caching mode. sane BIOS 应通过 MTRR 设置 regular memory(RAM) 为 cacheable, I/O memory 为 non-cacheable. memory type 冲突时, no matter between MTRR & PAT, or in MTRR itself, the conservative type will be chosen. 所以可以 deduce: 进入 Linux kernel 后，为了性能，kernel 希望所有可用 RAM 是 cacheable, 又因为可能的 type conflict, 为灵活使用 PAT, 所以希望 MTRR 设置的 caching mode 是 WB.

WTRR 和 E820 之间关系? E820_TYPE_RAM 表示 kernel 可用的 RAM 空间，kernel 需要它是 Write-Back 类型，若 buggy BIOS 未将可用 RAM 空间设置为 WB，则 trim 掉这部分空间。

```c
/* mtrr_trim_uncached_memory - trim RAM not covered by MTRRs
 * @end_pfn: ending page frame number
 *
 * Some buggy BIOSes don't setup the MTRRs properly for systems with certain
 * memory configurations.  This routine checks that the highest MTRR matches
 * the end of memory, to make sure the MTRRs having a write back type cover
 * all of the memory the kernel is intending to use.  If not, it'll trim any
 * memory off the end by adjusting end_pfn, removing it from the kernel's
 * allocation pools, warning the user with an obnoxious message. */
int __init mtrr_trim_uncached_memory(unsigned long end_pfn)
{
	unsigned long i, base, size, highest_pfn = 0, def, dummy;
	mtrr_type type;
	u64 total_trim_size;
	/* extra one for all 0 */
	int num[MTRR_NUM_TYPES + 1];

	/*
	 * Make sure we only trim memory on machines that
	 * support the Intel MTRR architecture:
	 */
	/* Skip self-documented code.
	 * Some noticeable points: 本函数仅针对 Intel MTRR Arch; Intel SDM 3a,
	 * 11.11.2.1 IA32_MTRR_DEF_TYPE MSR: Intel 推荐 default to UC, 省略代码中
	 * 有 check.
	 * 省略的代码还包括：
	 * 遍历 variable range MSR, 得到 region 的信息： base address, size, type;
	 * 找到 MTRR_TYPE_WRBACK region 中的最大 PFN, 看起来隐含含义: E820_TYPE_RAM
	 * 必为 MTRR_TYPE_WRBACK; Info: kvm/qemu 没有设置 MTRR, register value 为 0.
	 */

	/* Check entries number: */
	/* record number of regions of each memory type. */
	memset(num, 0, sizeof(num));
	for (i = 0; i < num_var_ranges; i++) {
		type = range_state[i].type;
		if (type >= MTRR_NUM_TYPES)
			continue;
		size = range_state[i].size_pfn;
		if (!size)
			type = MTRR_NUM_TYPES;
		num[type]++;
	}

	/* No entry for WB? */
	if (!num[MTRR_TYPE_WRBACK])
		return 0;

	/* Check if we only had WB and UC: 只有 WB & UC, 才会继续走下去? */
	if (num[MTRR_TYPE_WRBACK] + num[MTRR_TYPE_UNCACHABLE] !=
		num_var_ranges - num[MTRR_NUM_TYPES])
		return 0;

	memset(range, 0, sizeof(range));
	nr_range = 0;

	/* mtrr_tom2 related code is AMD specific, omit. */

	/* 1. Iterate to get all MTRR_TYPE_WRBACK from variable ranges(merge overlapped one);
	 * 2. 检查看 MTRR_TYPE_UNCACHABLE & MTRR_TYPE_WRPROT 的 range 是否和 1. 中结果
	 *    overlap, 若有，则 take out UC 和 WP range from WB, i.e., 从 1. 中结果减去
	 *    overlap range.  (仅限 1M 以上的 range, 1M 内是 fixed range.)
	 * 3. Sort range.
	 * nr_range 表示上述处理后的剩余 range number.
	 *
	 * Q: Why treat WRPROT as UNCACHEABLE in commit dd5523552c2897e3?
	 * A: Guess: write of Write-Protect is not cacheable. Refer to
	 *    "11.3 METHODS OF CACHING AVAILABLE" for definition of Write-Protect
	 */
	nr_range = x86_get_mtrr_mem_range(range, nr_range, 0, 0);

	/* Now we get all the sanitized variable ranges of type MTRR_TYPE_WRBACK
	 * from MTRR, trim E820 according to it, specifically, update E820_TYPE_RAM
	 * ranges not labeled with MTRR_TYPE_WRBACK to E820_TYPE_RESERVED.
	 * Skip the self-documented code.
	 */
	/* Check the head: */
	...

	/* Check the holes: */
	...

	/* Check the top: */
	...

	if (total_trim_size) {
		...
	}

	return 0;
}
```

## kernel_randomize_memory

KASLR 的主要工作：随机化 kernel 自身所在 physical & virtual address, 在 ZO 完成，并在 VO 入口代码对 page table 中 kernel 所在 range 进行修正，适配 ZO 中的随机化结果。 Besides，还有一部分 kernel 使用的数据虚拟空间(kaslr_regions[])需要随机化，这似乎?是最后的随机化处理。从 Documentation/x86/x86_64/mm.rst 中可以看出，kaslr_regions[] 列出的 range 后都有 reserve unused hole, 这看起来似乎就是为了 prepare for KASLR. 函数分析忽略 5-level paging 的情况。但此函数似乎只做了简单的数学计算，得到 region 随机化后的新 base address.

```c
/* Initialize base and padding for each memory region randomized with KASLR */
void __init kernel_randomize_memory(void)
{
	size_t i;
	unsigned long vaddr_start, vaddr;
	unsigned long rand, memory_tb;
	struct rnd_state rand_state;
	unsigned long remain_entropy;
	unsigned long vmemmap_size;

	/* 待随机化的 virtual address range 的起点 = direct mapping 的起始地址 */
	vaddr_start = pgtable_l5_enabled() ? __PAGE_OFFSET_BASE_L5 : __PAGE_OFFSET_BASE_L4;
	vaddr = vaddr_start;

	/*
	 * These BUILD_BUG_ON checks ensure the memory layout is consistent
	 * with the vaddr_start/vaddr_end variables. These checks are very
	 * limited....
	 */
	BUILD_BUG_ON(vaddr_start >= vaddr_end);
	BUILD_BUG_ON(vaddr_end != CPU_ENTRY_AREA_BASE);
	BUILD_BUG_ON(vaddr_end > __START_KERNEL_map);

	/* boot protocol.loadflags 中 bit 1: KASLR_FLAG 仅是 Kernel 内部使用，ZO 用它
	 * 来告诉 VO 关于 KASLR 的状态。这里就用到了。Refer: Documentation/x86/boot.rst */
	if (!kaslr_memory_enabled())
		return;

	/* 各 memory range 的定义都在 Documentation/x86/x86_64/mm.rst. Direct mapping
	 * 整个物理地址空间，所以是 MAX_PHYSMEM_BITS.*/
	kaslr_regions[0].size_tb = 1 << (MAX_PHYSMEM_BITS - TB_SHIFT);
	kaslr_regions[1].size_tb = VMALLOC_SIZE_TB;

	/* Update Physical memory mapping to available and
	 * add padding if needed (especially for memory hotplug support). */
	/* 看起来是说，为 memory hotplug 预留的虚拟地址空间，也要被随机化 */
	BUG_ON(kaslr_regions[0].base != &page_offset_base);
	memory_tb = DIV_ROUND_UP(max_pfn << PAGE_SHIFT, 1UL << TB_SHIFT) +
		CONFIG_RANDOMIZE_MEMORY_PHYSICAL_PADDING;

	/* Adapt phyiscal memory region size based on available memory */
	/* 术语使用实在让人困惑。memory_tb 由 max_pfn 计算得来，表示 RAM 的 size. 当我们说
	 * direct mapping 时，是指 RAM 空间的 page mapping, kaslr_regions[0].size_tb
	 * 初始化用的 MAX_PHYSMEM_BITS, 表示所有 CPU physical address space.
	 * 由于有 memory hole, 即 RAM 物理地址空间不是连续的，max_pfn 应该大于实际的 RAM size.
	 * 但看起来不 care memory hole, 就用 max_pfn 表示 RAM 最大地址。
	 * 现在, kaslr_regions[0].size_tb 表示 RAM size. */
	if (memory_tb < kaslr_regions[0].size_tb)
		kaslr_regions[0].size_tb = memory_tb;

	/* Calculate the vmemmap region size in TBs, aligned to a TB boundary. */
	/* 原来 vmemmap 空间是用来存放 struct page 数据结构的？ 描述 RAM size 的 size_tb
	 * 的 unit 由 TB 转为 page frame number. 看来 RAM page frame 与 struct page
	 * 是 1:1 映射关系。但 vmemmap 的 size 应该不超过 1TB? mm.rst 中预留的就是 1TB.
	 * 参考阅读：Documentation/vm/memory-model.rst */
	vmemmap_size = (kaslr_regions[0].size_tb << (TB_SHIFT - PAGE_SHIFT)) *
			sizeof(struct page);
	kaslr_regions[2].size_tb = DIV_ROUND_UP(vmemmap_size, 1UL << TB_SHIFT);

	/* Calculate entropy available between regions */
	/* KASLR 代码注释中常出现 "entropy", 不理解。就代码逻辑看，remain_entropy 表示：
	 * mm.rst 中给 kaslr_regions[] 预留的虚拟地址空间中，未被占用的部分，即剩余 size. */
	remain_entropy = vaddr_end - vaddr_start;
	for (i = 0; i < ARRAY_SIZE(kaslr_regions); i++)
		remain_entropy -= get_padding(&kaslr_regions[i]);

	/* Get one randomized number set. */
	prandom_seed_state(&rand_state, kaslr_get_random_long("Memory"));

	for (i = 0; i < ARRAY_SIZE(kaslr_regions); i++) {
		unsigned long entropy;

		/* Select a random virtual address using the extra entropy available. */
		/* entropy 以 byte 为单位表示一段未占用空间 size. */
		entropy = remain_entropy / (ARRAY_SIZE(kaslr_regions) - i);
		/* randomized number set => an unsigned long integer. */
		prandom_bytes_state(&rand_state, &rand, sizeof(rand));
		/* For 4-level paging, PUD_MASK 表示低 30 bit 是 0, 高位是 1. A PUD entry
		 * covers 1G space. Entropy 现在表示以 GB 为单位的 size.
		 * OK, now we get, 各 region 地址增加随机 GB. */
		entropy = (rand % (entropy + 1)) & PUD_MASK;
		vaddr += entropy; /* vaddr 表示当前 region 的 new base address. */
		*kaslr_regions[i].base = vaddr;

		/* Jump the region and add a minimum padding based on
		 * randomization alignment. */
		/* 这注释写的真像中国人。Region 的 new base + 它的 size 后向上对齐到 GB 边界，
		 * 即是 next region 的 base.*/
		vaddr += get_padding(&kaslr_regions[i]);
		vaddr = round_up(vaddr + 1, PUD_SIZE);
		remain_entropy -= entropy;
	}
}
```

## early_alloc_pgt_buf & reserve_brk

New piece of knowledge(for me): **.brk** section. LWN has an extended reading material for [special sections](https://lwn.net/Articles/531148/).  **.brk** section is defined in linker script as:

```
	__end_of_kernel_reserve = .;

	. = ALIGN(PAGE_SIZE);
	.brk : AT(ADDR(.brk) - LOAD_OFFSET) {
		__brk_base = .;
		. += 64 * 1024;		/* 64k alignment slop space */
		*(.brk_reservation)	/* areas brk users have reserved */
		__brk_limit = .;
	}

	. = ALIGN(PAGE_SIZE);		/* keep VO_INIT_SIZE page aligned */
	_end = .;
```

which looks simple, the input section must be **.brk.reservation**. **.brk** section locates at the end of kernel code image. The extra 64k space seems like guardian. Input section is defined via:
```c
/* Reserve space in the brk section.  The name must be unique within
 * the file, and somewhat descriptive.  The size is in bytes.  Must be
 * used at file scope.
 *
 * (This uses a temp function to wrap the asm so we can pass it the
 * size parameter; otherwise we wouldn't be able to.  We can't use a
 * "section" attribute on a normal variable because it always ends up
 * being @progbits, which ends up allocating space in the vmlinux
 * executable.) */
#define RESERVE_BRK(name,sz)						\
	static void __section(.discard.text) __used notrace		\
	__brk_reservation_fn_##name##__(void) {				\
		asm volatile (						\
			".pushsection .brk_reservation,\"aw\",@nobits;" \
			".brk." #name ":"				\
			" 1:.skip %c0;"					\
			" .size .brk." #name ", . - 1b;"		\
			" .popsection"					\
			: : "i" (sz));					\
	}
```

for who(in this case, early_alloc_pgt_buf()) want to reserve specific size space in brk section, just call this macro.

By this chance, finally understand the assembly directive: `.pushsection`, `.popsection`. Imagine a scenario: you are writing assembly, say, code belonging to .text, but suddenly you want to put following several lines of code into a different section, and then back to .text. Now it is time for`.pushsection` & `.popsection`. Take *RESERVE_BRK* above for example: push the current section *.discard.text* onto section stack, and replace it with *.brk_reservation* section, the following code will belong to it, then pop up the pushed section to resume.

看着代码，恍惚中又冒出一个问题：RESERVE_BRK 定义了 static 函数，函数中是 inline assembly，但函数没有被 reference, 其中的 assembly 怎么执行到的? 分析后发现是非常基础的问题。编译的过程是：pre-process, compile, assemble, link. Compile to get assembly file, 然后 assembler 会 parse 到这段代码，生成的 .o 文件中自然会有 .brk_reservation section; link 时按照 script 指导，将所有 .o 中的 brk section 输出到 vmlinux 的 .brk section.

但这里仍有一个 trick: "__used" 帮助实现目的：

	#define __used                          __attribute__((__used__))

参考 [GCC 文档](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-used-function-attribute)的解释:

>code must be emitted for the function even if it appears that the function is not referenced

内核编译使用 "-O2" 优化，RESERVE_BRK 定义的 static 函数没人调用，可想而知很可能会被优化掉，attribute `__used__` 使其免于被优化掉。

再看这段汇编代码，没有真正的 code, 仅是 label & directive: label 定义 symbol, 在符号表中可见； `.skip` 开辟入参 sz 指定的空间并填充 0; `.size` 将刚定义的 symbol 的 size 设置为 sz. **NOTE**: RESERVE_BRK 只是编译时在 **.brk** section 中 reserve 空间，这个设计使得通过 macro 预留 brk 空间很灵活。但这并不意味所有 reserve 空间都会使用，这里只是“规划/plan”一下。使用的事情要回头看 extend_brk().

```c
/*
 * By default need 3 4k for initial PMD_SIZE,  3 4k for 0-ISA_END_ADDRESS.
 * With KASLR memory randomization, depending on the machine e820 memory
 * and the PUD alignment. We may need twice more pages when KASLR memory
 * randomization is enabled.
 */
#ifndef CONFIG_RANDOMIZE_MEMORY
	#define INIT_PGD_PAGE_COUNT      6
#else
	#define INIT_PGD_PAGE_COUNT      12
#endif

#define INIT_PGT_BUF_SIZE	(INIT_PGD_PAGE_COUNT * PAGE_SIZE)

/* All above are non-sense to me for now, get back to it later.
 * Just get the PFN address for reserved space. */

RESERVE_BRK(early_pgt_alloc, INIT_PGT_BUF_SIZE);
void  __init early_alloc_pgt_buf(void)
{
	unsigned long tables = INIT_PGT_BUF_SIZE;
	phys_addr_t base;

	base = __pa(extend_brk(tables, PAGE_SIZE));

	/* The unit of these pgt_buf_* variables is PFN. */
	pgt_buf_start = base >> PAGE_SHIFT;
	pgt_buf_end = pgt_buf_start;
	pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);
}

/* Linker script defines __brk_base & __brk_limit to mark .brk section boundary,
 * but code use _brk_start & _brk_end in lieu of them.
 * 入参表明在这段空间申请的 size & alignment。 所以，规划的空间究竟使用多少，要看此
 * 函数被调用了多少次.
 */
void * __init extend_brk(size_t size, size_t align)
{
	size_t mask = align - 1;
	void *ret;

	/* _brk_start 在别处被赋值为 0，表明不再接受新的 brk space request. */
	BUG_ON(_brk_start == 0);
	/* alignment must be power of 2. */
	BUG_ON(align & mask);

	/* _brk_end 是上次 brk space request 后的结束地址，其 alignment 可能和现在不同。*/
	_brk_end = (_brk_end + mask) & ~mask;
	BUG_ON((char *)(_brk_end + size) > __brk_limit);

	/* 起始地址已对齐，then, just do it. */
	ret = (void *)_brk_end;
	_brk_end += size;

	memset(ret, 0, size);

	return ret;
}
```

如 reserve_brk 调用处的 comments 所说：need to conclude brk. The code is well self-documented.
```c
static void __init reserve_brk(void)
{
	if (_brk_end > _brk_start)
		memblock_reserve(__pa_symbol(_brk_start),
				 _brk_end - _brk_start);

	/* Mark brk area as locked down and no longer taking any
	   new allocations */
	_brk_start = 0;
}
```

## cleanup_highmap

(To be honest, the function comments is not friendly for me.)

Question: What is highmap? cleanup what?

highmap: for X86_64, the start virtual address of kernel is set to __START_KERNEL_map = 0xffffffff80000000, it is comparatively high in 64-bit virtual address space.

cleanup: virtual & physical address range of kernel are arranged to [__START_KERNEL_map, __START_KERNEL_map + KERNEL_IMAGE_SIZE] & [0, KERNEL_IMAGE_SIZE] respectively, with KERNEL_IMAGE_SIZE equals to 1G or 512M depends on KASLR enabled or not. In reality, the start address is aligned to *ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)*, and the actual kernel image size is far less than KERNEL_IMAGE_SIZE.

In head_64.S, page table construction has following code:

```
NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC, KERNEL_IMAGE_SIZE/PMD_SIZE)
```

(Refer to previous analysis for details, here is conclusion only)

which page mapping virtual address __START_KERNEL_map to physical address 0 with length KERNEL_IMAGE_SIZE. Obviously, the first several PMD entries is not mapped to kernel image itself because of alignment, i.e., invalid PMDs. In the meanwhile, although the actual memory image size of kernel is (**_end - _text**), the *.brk* section at the end may not be fully used, _brk_end denotes the actual end virtual address of kernel, so the page mapping of range [_brk_end, __START_KERNEL_map + KERNEL_IMAGE_SIZE] is meaningless too. The PMD entries of these 2 range are the subjects for cleanup.

With understanding above, it would be smooth to understand the function.

## init_mem_mapping

Let's go over the addressing range of X86_64 CPU first:

  1. for CPU support 4-level paging only, it has max virtual address width of 48(32+16) bits = 256 TB, and max physical address width of 46 bits = 64 TB, which just is the direct mapping size in Documentation/x86/x86_64/mm.rst.
  2. for CPU support 5-level paging, it has max virtual address width of 57(48+9) bits = 128 PB, and max physical address width of 52 bits = 4 PB, which is far less than 32 PB reserved by kernel for direct page mapping. WHY?

**PAGE_OFFSET**/**page_offset_base** denotes the start address used for direct mapping of all physical memory.

When probing page size, the **PSE** attracts my attention, which I didn't notice before, worth to know it in a separate sub-chapter.

Term distinguishing: when refering to **page**, it is of virtual address; when referring to **page frame**, it is of physical address.

In my view, logically, there are two important/hard piece of knowledge around this whole process:

  1. range split, which is analyzed via first call of init_memory_mapping().
  2. direct page mapping via init_top_pgd

```c
void __init init_mem_mapping(void)
{
	unsigned long end;

	pti_check_boottime_disable(); /* Not necessary, skip. */

	/* Probe the page sizes that supported by CPU.
	 * 4K page is the default for x86, no need to probe, it is also NOT marked
	 * explicitly in page_size_mask;
	 * X86_64 (in my opinion) support 2M page by default;
	 * 1G page depends on CPU model, which is determined jointly by X86_FEATURE_GBPAGES
	 * & CONFIG_X86_DIRECT_GBPAGES & kernel parameter "gbpages" or "nogbpages".
	 * direct_gbpages is decided by the latter two factors.
	 */
	probe_page_size_mask();

	/* Why do it here? */
	setup_pcid();

	/* end denotes the max physical address of RAM. */
#ifdef CONFIG_X86_64
	end = max_pfn << PAGE_SHIFT;
#else
	end = max_low_pfn << PAGE_SHIFT;
#endif

	/* the ISA range is always mapped regardless of memory holes */
	/* As mentioned above, the 64TB space starting from virtual address PAGE_OFFSET
	 * is used for(4-level paging) direct mapping CPU's physical address range.
	 *
	 * memory hole quick recalling: in the 1st 1M, the space after conventional
	 * memory(640k) is used for mapping ROMs, which is a memory hole.
	 *
	 * Involving a hard function: split_mem_range，even maintainer consider it
	 * to be garbage(https://lkml.org/lkml/2019/3/24/332). (It took me too much time.)
	 */
	init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL);

	/* Init the trampoline, possibly with KASLR memory offset */
	init_trampoline();

	/* Tip: analyze top_down first, as it is the default. After knowing top_down,
	 * bottom_up is a piece of cake.
	 */
	/*
	 * If the allocation is in bottom-up direction, we setup direct mapping
	 * in bottom-up, otherwise we setup direct mapping in top-down.
	 */
	if (memblock_bottom_up()) {
		unsigned long kernel_end = __pa_symbol(_end);

		/*
		 * we need two separate calls here. This is because we want to
		 * allocate page tables above the kernel. So we first map
		 * [kernel_end, end) to make memory above the kernel be mapped
		 * as soon as possible. And then use page tables allocated above
		 * the kernel to map [ISA_END_ADDRESS, kernel_end).
		 */
		memory_map_bottom_up(kernel_end, end);
		memory_map_bottom_up(ISA_END_ADDRESS, kernel_end);
	} else {
		memory_map_top_down(ISA_END_ADDRESS, end);
	}

#ifdef CONFIG_X86_64
	/* WHY??? */
	if (max_pfn > max_low_pfn) {
		/* can we preseve max_low_pfn? */
		max_low_pfn = max_pfn;
	}
#else
	early_ioremap_page_table_range_init();
#endif

	/* Aha, switch to new page global directory. */
	load_cr3(swapper_pg_dir);
	/* native_flush_tlb_local：reload CR3.
	 * native_flush_tlb_global: re-write CR4 with PGE on & off.
	 *
	 * Reference for flushing TLB: Intel SDM 3a, 4.10.4 Invalidation of TLBs and
	 * Paging-Structure Caches.
	 */
	__flush_tlb_all();

	x86_init.hyper.init_mem_mapping();

	/* Only available under CONFIG_MEMTEST. Seems quite time consuming. */
	early_memtest(0, max_pfn_mapped << PAGE_SHIFT);
}

/*
 * Tip: We could take 4Kb aligned parameters(start & end) for granted.  Actually,
 * memblock.memory is already trimmed to 4Kb aligned. The actual input parameter
 * is also 4Kb aligned.
 *
 * How to direct page mapping various ranges indicated by input parameter?
 * Strategy: use bigger page size as possible as it can. (X86_64 has 4k, 2M, 1G)
 * Tip: the address of 4Kb page must be 4Kb aligned, so does 2Mb & 1Gb page.
 *
 * The 1st step is to split RAM range into several segments via split_mem_range(),
 * then mapping.
 *
 * 初看 split_mem_range 几遍仍不懂，遂 google 函数名，有所得，但仍不懂。可见 importance
 * of narrative ability. 下有详细分析其 split memory range 机制。
 */
/*
 * Setup the direct mapping of the physical memory at PAGE_OFFSET.
 * This runs before bootmem is initialized and gets pages directly from
 * the physical memory. To access them they are temporarily mapped.
 */
unsigned long __ref init_memory_mapping(unsigned long start,
					       unsigned long end)
{
	/* Why NR_RANGE_MR = 5 under x86_64? Answer in analysis of split_mem_range. */
	struct map_range mr[NR_RANGE_MR];
	unsigned long ret = 0;
	int nr_range, i;

	/* Normally, log of pr_debug doesn't appear in dmesg, unless:
	 *   1. CONFIG_DYNAMIC_DEBUG=y, &
	 *   2. file compiled with -DDEBUG: CFLAGS_[filename].o := -DDEBUG
	 *      (https://www.kernel.org/doc/local/pr_debug.txt)
	 */
	pr_debug("init_memory_mapping: [mem %#010lx-%#010lx]\n",
	       start, end - 1);

	memset(mr, 0, sizeof(mr));
	/* 之前理解的困惑点：不知道 init_memory_mapping 会被多次调用，函数中加打印后才知道。
	 * 费了好大劲儿终于看懂这个函数TAT。 Details at below. */
	nr_range = split_mem_range(mr, 0, start, end);

	for (i = 0; i < nr_range; i++)
		ret = kernel_physical_mapping_init(mr[i].start, mr[i].end,
						   mr[i].page_size_mask);

	/* Record the mapped pfn above. Details at below. */
	add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT);

	return ret >> PAGE_SHIFT;
}

/* Split the input range with following strategy:
 *
 *     1. range [start - 1st 2Mb boundary] use 4Kb page size;
 *     2. range [1st 2Mb boundary - 1st 1Gb boundary] use 2Mb page size;
 *     3. range [1st 1Gb boundary - last 1Gb boundary] use 1Gb page size;
 *     4. range [last 1Gb boundary - last 2Mb boundary] use 2Mb page size;
 *     5. range [last 2Mb boundary - end] use 4kb page size;
 *
 * There are 5 (NR_RANGE_MR) memory ranges at most.
 *
 * An enlightening Q&A: https://www.spinics.net/lists/linux-mm/msg148870.html
 * which reveals the fact: 2Mb page must be 2M aligned.
 *
 * Take an example to illustrate the strategy above, say a range with starting
 * address 5Mb, size (2G + 2M + 4K), i.e., end address = start + size = 2G + 7M
 * + 4K, according to the strategy:
 *     - Range [5M - 6M] is set to use 4K page size.
 *     - Range [6M - 1G] is set to use 2M page size.
 *     - Range [1G - 2G] is set to use 1G page size
 *     - Range [2G - 2G+6M] set to use 2M page size.
 *     - Range [2G+6M - 2G+1M+4K] set to use 4K page size.
 *
 * Actually, the input range is aligned to 4Kb.
 */
static int __meminit split_mem_range(struct map_range *mr, int nr_range,
				     unsigned long start,
				     unsigned long end)
{
	unsigned long start_pfn, end_pfn, limit_pfn;
	unsigned long pfn;
	int i;

	/* In my example, limit_pfn = 0x80701.
	 *
	 * many pfn variables, let's make them clear:
	 *   limit_pfn denotes the upper boundary of input range. 用来做判断；
	 *   start_pfn & end_pfn are used for marking the boundary of each split range;
	 *   The austere "pfn" is just for storing calculating result.
	 */
	limit_pfn = PFN_DOWN(end);

	/* head if not big page alignment ? */
	pfn = start_pfn = PFN_DOWN(start);
#ifdef CONFIG_X86_32
	/* SKIP */
#else /* CONFIG_X86_64 */
	/* 1st 2Mb boundary: address 0x600000(6M) in my example. */
	end_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE));
#endif
	if (end_pfn > limit_pfn)
		end_pfn = limit_pfn;
	if (start_pfn < end_pfn) {
		/* 4k page size is the default, no need to mark it explicitly. */
		nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0);
		pfn = end_pfn;
	}

	/* big page (2M) range */
	start_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE));
#ifdef CONFIG_X86_32
	/* SKIP */
#else /* CONFIG_X86_64 */
	/* 1st 1Gb boundary */
	end_pfn = round_up(pfn, PFN_DOWN(PUD_SIZE));
	if (end_pfn > round_down(limit_pfn, PFN_DOWN(PMD_SIZE)))
		end_pfn = round_down(limit_pfn, PFN_DOWN(PMD_SIZE));
#endif

	if (start_pfn < end_pfn) {
		nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
				page_size_mask & (1<<PG_LEVEL_2M));
		pfn = end_pfn;
	}

#ifdef CONFIG_X86_64
	/* big page (1G) range */
	start_pfn = round_up(pfn, PFN_DOWN(PUD_SIZE)); /* 1st 1Gb boundary. */
	end_pfn = round_down(limit_pfn, PFN_DOWN(PUD_SIZE)); /* last 1Gb boundary. */
	if (start_pfn < end_pfn) {
		nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
				page_size_mask &
				 ((1<<PG_LEVEL_2M)|(1<<PG_LEVEL_1G)));
		pfn = end_pfn;
	}

	/* tail is not big page (1G) alignment */
	start_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE)); /* equals last 1Gb boundary. */
	end_pfn = round_down(limit_pfn, PFN_DOWN(PMD_SIZE)); /* last 2Mb boundary. */
	if (start_pfn < end_pfn) {
		nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
				page_size_mask & (1<<PG_LEVEL_2M));
		pfn = end_pfn;
	}
#endif

	/* tail is not big page (2M) alignment */
	/* The rest range use 4Kb page size. page frame is innately 4Kb aligned, no need
	 * to round up/down. */
	start_pfn = pfn;
	end_pfn = limit_pfn;
	nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0);

	/*
	 * Check if a split range support bigger page size, and mark it if so.
	 * How: expand the range to bigger page size boundary, if expanded range still
	 * belongs to memblock.memory, then it could use bigger page size. (Lack of sth?)
	 *
	 * Assuming a range [5M - 2G], split range [5M - 6M] is set to use 4Kb page,
	 * if [4Mb - 6Mb] belongs to memblock.memory, then mark [5M - 6M] to be mapped
	 * with 2Mb page. BUT, can range [5M - 6M] be mapped with 2Mb page, to be dug deeper.
	 *
	 * Finally, find the answer to these confusion: these functions are aiming to setup
	 * the page structures for direct mapping, i.e., fill the entries in page structure.
	 * Numerally, physical range [5M - 6M] can't be mapped with 2M page size, but if
	 * [4M - 6M] belongs to RAM(means it can be mapped with 2M page size), the page table
	 * entry for this 2M  memory can be filled. Just from the perspective of filling
	 * entries of page structure, the logic has no problem.
	 */
	if (!after_bootmem)
		adjust_range_page_size_mask(mr, nr_range);

	/* Following comments is too Chinglish, should be:
	 *
	 *     try to merge contiguous range with the same page size mask.
	 */
	/* try to merge same page size and continuous */
	for (i = 0; nr_range > 1 && i < nr_range - 1; i++) {
		unsigned long old_start;
		if (mr[i].end != mr[i+1].start ||
		    mr[i].page_size_mask != mr[i+1].page_size_mask)
			continue;
		/* move it */
		old_start = mr[i].start;
		memmove(&mr[i], &mr[i+1],
			(nr_range - 1 - i) * sizeof(struct map_range));
		mr[i--].start = old_start;
		nr_range--;
	}

	for (i = 0; i < nr_range; i++)
		pr_debug(" [mem %#010lx-%#010lx] page %s\n",
				mr[i].start, mr[i].end - 1,
				page_size_string(&mr[i]));

	return nr_range;
}
```

Range split can conclude now. As said above, another important piece is direct page mapping with **init_top_pgt**, analyze it via memory_map_top_down(), which is the traditional way of direct page mapping.

Tip:

  1. There are memory holes in physical address, only RAM is direct page mapped, holes are skipped.
  2. memblock_find_in_range() allocates memory in top-down or bottom-up way, which is hidden in the function, and can't tell by function name.
  3. commit 8d57470d8f8596 gives many pieces of extra information.

```c
static void __init memory_map_top_down(unsigned long map_start,
				       unsigned long map_end)
{
	unsigned long real_end, last_start;
	unsigned long step_size;
	unsigned long addr;
	unsigned long mapped_ram_size = 0;

	/* Alignment is applied on start address. PMD_SIZE(2M) alignment implies there
	 * may be a gap between (real_end, end), because 4K alignment is always enforced,
	 * but not 2M.
	 */
	/* xen has big range in reserved near end of ram, skip it at first.*/
	addr = memblock_find_in_range(map_start, map_end, PMD_SIZE, PMD_SIZE);
	real_end = addr + PMD_SIZE;

	/* The first several pieces of 4Kb memory for page structure is from BRK buffer. */
	/* step_size need to be small so pgt_buf from BRK could cover it */
	step_size = PMD_SIZE;
	max_pfn_mapped = 0; /* will get exact value next */
	min_pfn_mapped = real_end >> PAGE_SHIFT;
	last_start = real_end;

	/*
	 * We start from the top (end of memory) and go to the bottom.
	 * The memblock_find_in_range() gets us a block of RAM from the
	 * end of RAM in [min_pfn_mapped, max_pfn_mapped) used as new pages
	 * for page table.
	 */
	while (last_start > map_start) {
		unsigned long start;

		/* This is tricky to compare *address* with *size*.  */
		if (last_start > step_size) {
			start = round_down(last_start - 1, step_size);
			if (start < map_start)
				start = map_start;
		} else
			start = map_start;
		mapped_ram_size += init_range_memory_mapping(start,
							last_start);
		last_start = start;
		min_pfn_mapped = last_start >> PAGE_SHIFT;
		if (mapped_ram_size >= step_size)
			step_size = get_new_step_size(step_size);
	}

	if (real_end < map_end)
		init_range_memory_mapping(real_end, map_end);
}

static unsigned long __init init_range_memory_mapping(
					   unsigned long r_start,
					   unsigned long r_end)
{
	unsigned long start_pfn, end_pfn;
	unsigned long mapped_ram_size = 0;
	int i;

	/* clamp operation ensures memory holes is omitted. */
	for_each_mem_pfn_range(i, MAX_NUMNODES, &start_pfn, &end_pfn, NULL) {
		u64 start = clamp_val(PFN_PHYS(start_pfn), r_start, r_end);
		u64 end = clamp_val(PFN_PHYS(end_pfn), r_start, r_end);
		/* two ranges don't overlap. */
		if (start >= end)
			continue;

		/*
		 * if it is overlapping with brk pgt, we need to
		 * alloc pgt buf from memblock instead.
		 */
		/* Really confusing at first: why mapped physical range can't overlap
		 * with BRK buffer? c9b3234a6abad says it is for a XEN bug.
		 */
		can_use_brk_pgt = max(start, (u64)pgt_buf_end<<PAGE_SHIFT) >=
				    min(end, (u64)pgt_buf_top<<PAGE_SHIFT);
		init_memory_mapping(start, end, PAGE_KERNEL);
		mapped_ram_size += end - start;
		can_use_brk_pgt = true;
	}

	return mapped_ram_size;
}
```

Since range split is already analyzed above, skip to kernel_physical_mapping_init:

```c
/*
 * Create page table mapping for the physical memory for specific physical
 * addresses. Note that it can only be used to populate non-present entries.
 * The virtual and physical addresses have to be aligned on PMD level
 * down. It returns the last physical address mapped.
 */
unsigned long __meminit
kernel_physical_mapping_init(unsigned long paddr_start,
			     unsigned long paddr_end,
			     unsigned long page_size_mask)
{
	return __kernel_physical_mapping_init(paddr_start, paddr_end,
					      page_size_mask, true);
}

/* Note: top page directory used by init_mm is swapper_pg_dir, which is
 * init_top_pgt in turn, check the geometry in:
 *   https://github.com/PinoTsao/Memo/blob/master/boot/earlymapping.txt
 *
 * init_top_pgt is initialized at the bottom of x86_64_start_kernel(), it has only one
 * entry valid, which is the last one, IIRC.
 *
 * page mapping code is complicated, just grab the main idea for now:
 * Page structures is initialized from PTE to PGD. Allocate 4k memory for
 * page structure if necessary, fill out the necessary entries, then write table's
 * physical address to corresponding entry of its higher level table.
 */
static unsigned long __meminit
__kernel_physical_mapping_init(unsigned long paddr_start,
			       unsigned long paddr_end,
			       unsigned long page_size_mask,
			       bool init)
{
	bool pgd_changed = false;
	unsigned long vaddr, vaddr_start, vaddr_end, vaddr_next, paddr_last;

	/* __va() returns the direct mapping virtual address of PAGE_OFFSET for
	 * physical address.
	 */
	paddr_last = paddr_end;
	vaddr = (unsigned long)__va(paddr_start);
	vaddr_end = (unsigned long)__va(paddr_end);
	vaddr_start = vaddr;

	for (; vaddr < vaddr_end; vaddr = vaddr_next) {
		/* Get the PGD entry in init_top_pgt for vaddr.
		 * Tip: A PGD entry maps 512G in 4-level paging.
		 */
		pgd_t *pgd = pgd_offset_k(vaddr);
		p4d_t *p4d;

		/* Get the virtual address of next PGD entry. */
		vaddr_next = (vaddr & PGDIR_MASK) + PGDIR_SIZE;

		if (pgd_val(*pgd)) {
			p4d = (p4d_t *)pgd_page_vaddr(*pgd);
			paddr_last = phys_p4d_init(p4d, __pa(vaddr),
						   __pa(vaddr_end),
						   page_size_mask,
						   init);
			continue;
		}

		p4d = alloc_low_page();
		paddr_last = phys_p4d_init(p4d, __pa(vaddr), __pa(vaddr_end),
					   page_size_mask, init);

		/* functions *_populate_init() are defined via macro: DEFINE_POPULATE. When
		 * initializing the page table, they are actually *_populate_safe()
		 */
		spin_lock(&init_mm.page_table_lock);
		if (pgtable_l5_enabled())
			pgd_populate_init(&init_mm, pgd, p4d, init);
		else
			p4d_populate_init(&init_mm, p4d_offset(pgd, vaddr),
					  (pud_t *) p4d, init);

		spin_unlock(&init_mm.page_table_lock);
		pgd_changed = true;
	}

	if (pgd_changed)
		/* Refer to its comments. */
		sync_global_pgds(vaddr_start, vaddr_end - 1);

	return paddr_last;
}

/*
 * Comments confused me for quite while: WHY returned pages are already directly mapped?
 * Fact: first several 4K memory is from BRK buffer, which is page mapped as a part of
 * kernel itself, which is also direct mapping. But in Documentation/x86/x86_64/mm.rst,
 * "direct mapping" is dedicated for offset PAGE_OFFSET, which is the reason leads to
 * confusion.  Direct mapping means, virtual address = physical address + fixed offset.
 * Commit 8d57470d8f8596 provides many useful information.
 *
 * Currently, kernel is using early_top_pgt rather than init_top_pgt. alloc_low_pages()
 * returned the direct mapping address of PAGE_OFFSET, and caller use it to write page
 * structure entries, for BRK buffer virtual address, early_top_pgt doesn't have related
 * page structure entries, but it doesn't bother since page fault exception handler will
 * deal with it(creating corresponding entries).
 *
 * The trick: page structures are initialized in reverse order, i.e., from PTE
 * to PGD, and PUD's physical address is written into its PGD entry in init_top_pgt
 * in the last minute.
 */
/*
 * Pages returned are already directly mapped.
 *
 * Changing that is likely to break Xen, see commit:
 *
 *    279b706 x86,xen: introduce x86_init.mapping.pagetable_reserve
 *
 * for detailed information.
 */
__ref void *alloc_low_pages(unsigned int num)
{
	unsigned long pfn;
	int i;

	if (after_bootmem) {
		unsigned int order;

		order = get_order((unsigned long)num << PAGE_SHIFT);
		return (void *)__get_free_pages(GFP_ATOMIC | __GFP_ZERO, order);
	}

	/* 1st mapping range (0 - 1M) need 3 tables: PUD PMD PTE */

	if ((pgt_buf_end + num) > pgt_buf_top || !can_use_brk_pgt) {
		unsigned long ret = 0;

		if (min_pfn_mapped < max_pfn_mapped) {
			/* The range is already directly page mapped. */
			ret = memblock_find_in_range(
					min_pfn_mapped << PAGE_SHIFT,
					max_pfn_mapped << PAGE_SHIFT,
					PAGE_SIZE * num , PAGE_SIZE);
		}
		if (ret)
			memblock_reserve(ret, PAGE_SIZE * num);
		else if (can_use_brk_pgt)
			ret = __pa(extend_brk(PAGE_SIZE * num, PAGE_SIZE));

		if (!ret)
			panic("alloc_low_pages: can not alloc memory");

		pfn = ret >> PAGE_SHIFT;
	} else {
		pfn = pgt_buf_end;
		pgt_buf_end += num;
	}

	for (i = 0; i < num; i++) {
		void *adr;

		adr = __va((pfn + i) << PAGE_SHIFT);
		clear_page(adr);
	}

	return __va(pfn << PAGE_SHIFT);
}

struct range pfn_mapped[E820_MAX_ENTRIES];
int nr_pfn_mapped;

/* Record the page-mapped page frame into pfn_mapped[]. */
static void add_pfn_range_mapped(unsigned long start_pfn, unsigned long end_pfn)
{
	/* Details at below. */
	nr_pfn_mapped = add_range_with_merge(pfn_mapped, E820_MAX_ENTRIES,
					     nr_pfn_mapped, start_pfn, end_pfn);

	/*
	 * Get rid of invalid range which has end = 0, and sort the array.
	 * clean algorithm: 从前遍历找到第一个 invalid range(end = 0), 从后遍历找到第一个
	 * valid range (end != 0), swap & clear the invalid range(make start = 0)
	 * return valid range number. "nr_pfn_mapped" seems to be a misnomer.
	 */
	nr_pfn_mapped = clean_sort_range(pfn_mapped, E820_MAX_ENTRIES);

	/* Finally see the initialization of these two "max mapped pfn" variables.
	 * max_low_pfn_mapped has only 1 user: ACPI.
	 */
	max_pfn_mapped = max(max_pfn_mapped, end_pfn);

	if (start_pfn < (1UL<<(32-PAGE_SHIFT)))
		max_low_pfn_mapped = max(max_low_pfn_mapped,
					 min(end_pfn, 1UL<<(32-PAGE_SHIFT)));
}

int add_range_with_merge(struct range *range, int az, int nr_range,
		     u64 start, u64 end)
{
	int i;

	if (start >= end)
		return nr_range;

	/* loop to determining whether there is overlap between new PFN range
	 * & existed PFN ranges in pfn_mapped[] or not, merge them if yes.
	 */
	/* get new start/end: */
	for (i = 0; i < nr_range; i++) {
		u64 common_start, common_end;

		/* empty array element. */
		if (!range[i].end)
			continue;

		/* Tricky, (common_start > common_end) means no overlap. */
		common_start = max(range[i].start, start);
		common_end = min(range[i].end, end);
		if (common_start > common_end)
			continue;

		/* new start/end, will add it back at last */
		start = min(range[i].start, start);
		end = max(range[i].end, end);

		/* There *is* overlap, drop the existing overlapped range, because the new
		 * merged range is already got, and wait to be add to the array */
		memmove(&range[i], &range[i + 1],
			(nr_range - (i + 1)) * sizeof(range[i]));

		/* clear to mark the end of array. */
		range[nr_range - 1].start = 0;
		range[nr_range - 1].end   = 0;
		nr_range--;
		i--;
	}

	/* Need to add it: */
	return add_range(range, az, nr_range, start, end);
}
```

In direct page mapping, there is another important piece of knowledge worth to know, which is in memory_map_bottom_up(). After knowing well about memory_map_top_down(), bottom_up will be a piece of case. Only 2 piece of tips:

  1. Comments inside have copy&paste error: not *end*, but *bottom*
  2. *if (step_size && map_end - start > step_size)* fix the corresponding tricky point mentioned in top_down function.

### PSE & PSE-36

[Page Size Extension(PSE)](https://en.wikipedia.org/wiki/Page_Size_Extension) & [PSE-36](https://en.wikipedia.org/wiki/PSE-36) are X86 CPU features, which are often mentioned when involving with paging. They may confuse people when not take a serious look. That is what is gonna made clear.

Originally, 32-bit X86 CPU only supports 4Kb page with address capability of 32-bit for both virtual address and physical address. Soon, people have [motivation](https://en.wikipedia.org/wiki/Page_Size_Extension#Motivation) to expand it, then 4Mb page support comes out, and the scheme is simple & no need to illustrate here. But still, the physical address space is limited to 4Gb(32-bit), which motivate more wide physical address, which results in PSE-36.

**36** of PSE-36 means to expand the physical address width from 32-bit to 36-bit, later it was even expanded to [40-bit](https://en.wikipedia.org/wiki/PSE-36#Extension_up_to_40_bits), but still use **PSE-36** to denote this feature. **PSE** has a separate control bit in CR4, while **PSE-36** doesn't, which only can be detected by CPUID. Actually CPU who support PSE-36 is not necessary to have 36-bit physical address exactly, the exact expansion bits can be confirmed via CPUID. In the following analysis will still use 36-bit physical address as the example.

When CPU support PSE-36, linear address space is still 32-bit with physical address space expanded to 36-bit, the page directory entry structure is illustrated as in [Wikipedia](https://en.wikipedia.org/wiki/PSE-36#Activation_and_use) & *Figure 4-4. Formats of CR3 and Paging-Structure Entries with 32-Bit Paging* of Intel SDM 3a, which also implies, 4Kb granularity page is only available under 4Gb, because the bits in the middle of the structure is used for indexing the page table to find the 4Kb page, which also implies the highest address bit(M - 32) of a page frame will be 0.

## idt_setup_early_pf

As the name tells, install a new page fault handler. But the definition of handler **asm_exc_page_fault** is tricky.

```c
/*
 * Early traps running on the DEFAULT_STACK because the other interrupt
 * stacks work only after cpu_init().
 */
static const __initconst struct idt_data early_pf_idts[] = {
	INTG(X86_TRAP_PF,		asm_exc_page_fault),
};
```

A simple search won't find definition of **asm_exc_page_fault**. Intuition tells me searching **exc_page_fault** may work, and yes it does:

In *arch/x86/include/asm/idtentry.h*:
```c
DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF,	exc_page_fault);

#define DECLARE_IDTENTRY_RAW_ERRORCODE(vector, func)			\
	DECLARE_IDTENTRY_ERRORCODE(vector, func)

#ifndef __ASSEMBLY__
	#define DECLARE_IDTENTRY_ERRORCODE(vector, func)			\
		asmlinkage void asm_##func(void);				\
		asmlinkage void xen_asm_##func(void);				\
		__visible void func(struct pt_regs *regs, unsigned long error_code)
#else
	#define DECLARE_IDTENTRY_ERRORCODE(vector, func)			\
		idtentry vector asm_##func func has_error_code=1
#endif
```

In *arch/x86/mm/fault.c*:
```c
DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
	...
}
```

But still don't know the definition, trying search **idtentry** and find in *arch/x86/entry/entry_64.S*:

```assembly
.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	...
SYM_CODE_END(\asmsym)
.endm
```

It involves some basic knowledge to finally find how **asm_exc_page_fault** is defined. Check file: *arch/x86/entry/.entry_64.o.cmd* & *arch/x86/mm/.fault.o.cmd*, both include *arch/x86/include/asm/idtentry.h*. The pre-processed *fault.c* will be something like:

```c
DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF,	exc_page_fault);

#define DECLARE_IDTENTRY_RAW_ERRORCODE(vector, func)			\
	DECLARE_IDTENTRY_ERRORCODE(vector, func)

#define DECLARE_IDTENTRY_ERRORCODE(vector, func)			\
	asmlinkage void asm_##func(void);				\
	asmlinkage void xen_asm_##func(void);				\
	__visible void func(struct pt_regs *regs, unsigned long error_code)

#define DEFINE_IDTENTRY_RAW_ERRORCODE(func)				\
__visible noinstr void func(struct pt_regs *regs, unsigned long error_code)

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
	...
}
```

which declares **asm_exc_page_fault** & defined **exc_page_fault**. The pre-processed *entry_64.S* will be something like:

```
DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF,	exc_page_fault);

#define DECLARE_IDTENTRY_RAW_ERRORCODE(vector, func)			\
	DECLARE_IDTENTRY_ERRORCODE(vector, func)

#define DECLARE_IDTENTRY_ERRORCODE(vector, func)			\
	idtentry vector asm_##func func has_error_code=1

.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	...
SYM_CODE_END(\asmsym)
.endm
```

which defines **asm_exc_page_fault**, and linker will find it when linking.

**idt_setup_early_pf** is pretty simple by itself, it only worth to take a look at the descriptor in **early_pf_idts** which is also simple after have the background knowledge from Intel SDM. The interesting part for now is that how page fault handler is executed.(Not include page mapping details)

```c
.macro idtentry_body cfunc has_error_code:req

	call	error_entry
	UNWIND_HINT_REGS

	movq	%rsp, %rdi			/* pt_regs pointer into 1st argument*/

	.if \has_error_code == 1
		movq	ORIG_RAX(%rsp), %rsi	/* get error code into 2nd argument*/
		movq	$-1, ORIG_RAX(%rsp)	/* no syscall to restart */
	.endif

	call	\cfunc

	jmp	error_return
.endm

.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code*8
	ASM_CLAC

	.if \has_error_code == 0
		pushq	$-1			/* ORIG_RAX: no syscall to restart */
	.endif

	.if \vector == X86_TRAP_BP
		/*
		 * If coming from kernel space, create a 6-word gap to allow the
		 * int3 handler to emulate a call instruction.
		 */
		testb	$3, CS-ORIG_RAX(%rsp)
		jnz	.Lfrom_usermode_no_gap_\@
		.rept	6
		pushq	5*8(%rsp)
		.endr
		UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
	.endif

	idtentry_body \cfunc \has_error_code

_ASM_NOKPROBE(\asmsym)
SYM_CODE_END(\asmsym)
.endm

error_entry)
	UNWIND_HINT_FUNC
	cld
	PUSH_AND_CLEAR_REGS save_ret=1
	ENCODE_FRAME_POINTER 8
	testb	$3, CS+8(%rsp)
	jz	.Lerror_kernelspace

	/*
	 * We entered from user mode or we're pretending to have entered
	 * from user mode due to an IRET fault.
	 */
	SWAPGS
	FENCE_SWAPGS_USER_ENTRY
	/* We have user CR3.  Change to kernel CR3. */
	SWITCH_TO_KERNEL_CR3 scratch_reg=%rax

.Lerror_entry_from_usermode_after_swapgs:
	/* Put us onto the real thread stack. */
	popq	%r12				/* save return addr in %12 */
	movq	%rsp, %rdi			/* arg0 = pt_regs pointer */
	call	sync_regs
	movq	%rax, %rsp			/* switch stack */
	ENCODE_FRAME_POINTER
	pushq	%r12
	ret

.Lerror_entry_done_lfence:
	FENCE_SWAPGS_KERNEL_ENTRY
.Lerror_entry_done:
	ret

	/*
	 * There are two places in the kernel that can potentially fault with
	 * usergs. Handle them here.  B stepping K8s sometimes report a
	 * truncated RIP for IRET exceptions returning to compat mode. Check
	 * for these here too.
	 */
.Lerror_kernelspace:
	leaq	native_irq_return_iret(%rip), %rcx
	cmpq	%rcx, RIP+8(%rsp)
	je	.Lerror_bad_iret
	movl	%ecx, %eax			/* zero extend */
	cmpq	%rax, RIP+8(%rsp)
	je	.Lbstep_iret
	cmpq	$.Lgs_change, RIP+8(%rsp)
	jne	.Lerror_entry_done_lfence

	/*
	 * hack: .Lgs_change can fail with user gsbase.  If this happens, fix up
	 * gsbase and proceed.  We'll fix up the exception and land in
	 * .Lgs_change's error handler with kernel gsbase.
	 */
	SWAPGS
	FENCE_SWAPGS_USER_ENTRY
	jmp .Lerror_entry_done

.Lbstep_iret:
	/* Fix truncated RIP */
	movq	%rcx, RIP+8(%rsp)
	/* fall through */

.Lerror_bad_iret:
	/*
	 * We came from an IRET to user mode, so we have user
	 * gsbase and CR3.  Switch to kernel gsbase and CR3:
	 */
	SWAPGS
	FENCE_SWAPGS_USER_ENTRY
	SWITCH_TO_KERNEL_CR3 scratch_reg=%rax

	/*
	 * Pretend that the exception came from user mode: set up pt_regs
	 * as if we faulted immediately after IRET.
	 */
	mov	%rsp, %rdi
	call	fixup_bad_iret
	mov	%rax, %rsp
	jmp	.Lerror_entry_from_usermode_after_swapgs
SYM_CODE_END(error_entry)

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
	...
}
```

## reserve_initrd

Get the direct page mapped address of initrd in *initrd_start* & *initrd_end* for future use since direct page mapping has been done.

	Note: in the begaining of setup_arch(), early_reserve_initrd() has
	reserved ramdisk space in memblock.reserved.

```c
#ifdef CONFIG_BLK_DEV_INITRD
static void __init reserve_initrd(void)
{
	/* Assume only end is not page aligned */
	/* 从 bootparam 中拿到 ramdisk 的 physical address & size. */
	u64 ramdisk_image = get_ramdisk_image();
	u64 ramdisk_size  = get_ramdisk_size();
	u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);
	u64 mapped_size;

	if (!boot_params.hdr.type_of_loader ||
	    !ramdisk_image || !ramdisk_size)
		return;		/* No initrd provided by bootloader */

	initrd_start = 0;

	printk(KERN_INFO "RAMDISK: [mem %#010llx-%#010llx]\n", ramdisk_image,
			ramdisk_end - 1);

	/* init_mem_mapping() has recorded direct page mapped PFN ranges, if ramdisk space
	 * exits there，it can be accessed directly by virtual address returned by __va(),
	 * which is the ease case.
	 */
	if (pfn_range_is_mapped(PFN_DOWN(ramdisk_image),
				PFN_DOWN(ramdisk_end))) {
		/* All are mapped, easy case */
		initrd_start = ramdisk_image + PAGE_OFFSET;
		initrd_end = initrd_start + ramdisk_size;
		return;
	}

	/* Or else, 在 memblock.memory 中分配物理空间，将 ramdisk copy 过来。这分配的空间
	 * 自然不能再被 kernel 任意使用，所以 mark it in memblock.reserved. 同样记录 newly
	 * allocated 空间的虚拟地址。*/
	relocate_initrd();

	/* So, original ramdisk space can be freed. */
	memblock_free(ramdisk_image, ramdisk_end - ramdisk_image);
}
#else
	...
#endif
```

## acpi_boot_table_init

This is the first ACPI initialization function out of several one. **boot** indicates this is initialization during OS booting. **table** indicates this is initialization is for ACPI  table, in other words, transform the pristine ACPI table into style of kernel itself.

So there should be other later initialization semantically, what's the difference? TBD.

```c
acpi_boot_table_init()
{
	/* Specific type of PC doesn't support ACPI or support it well, which are
	 * blacklisted in acpi_dmi_table. In case Linux running on these platform,
	 * the corresponding platform quirks must be executed first, which is actually
	 * disable ACPI or part of its functions.
	 */
	dmi_check_system(acpi_dmi_table);

	/* If acpi_disabled, bail out */
	/* 若 ACPI 功能被禁用, 就不需 care 数量庞大的 ACPI 代码，省事儿了，this is the
	 * meaning of "bail out" 的含义：纾困。
	 *
	 * In which case ACPI will be disabled?
	 *  1. Specific platform: 1st item of acpi_dmi_table.
	 *  2. kernel parameter: acpi=off
	 */
	if (acpi_disabled)
		return;

	/* Initialize the ACPI boot-time table parser. */
	if (acpi_table_init()) {
		disable_acpi();
		return;
	}

	/* Simple Boot Flag Table is a industry spec defined elsewhere, which is simply
	 * reserved by ACPI. (下笔时最新版本是 2.1)
	 *
	 * Quick knowledge:
	 * On PC-AT BIOS computer, a BOOT register is defined in main CMOS memory.
	 * (typically accessed through I/O ports 70h/71h on Intel Architecture platforms)
	 * The location of the BOOT register in CMOS is indicated by a BOOT table
	 * found via the ACPI RSDT table.
	 * 一个 OS 与 firmware 交换信息的机制。(待确认：在 CMOS 中的 1-byte register 写数据,
	 * firmware 运行时检查其中的 bit.)
	 *
	 * Handler just to get the I/O ports.
	 */
	acpi_table_parse(ACPI_SIG_BOOT, acpi_parse_sbf);

	/* blacklist may disable ACPI entirely */
	/* acpi_dmi_table above lists a group of platforms who knowingly don't support ACPI
	 * or support it well. While acpi_blacklist lists the specific revision of specific
	 * table of specific platform who has serious bugs which need to disable ACPI totally,
	 * so it is going to check against ACPI tables which is parsed just before.
	 *
	 * Besides, there are also other ACPI compatibility check: ACPI OSI, ACPI REV override,
	 * these are not serious bugs which can be work around. TBD.
	 */
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

/* In driver/acpi/table.c. */
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

	/* ACPI driver use struct acpi_table_desc to describe an ACPI table, it statically
	 * allocates a root table array, which is managed by acpi_gbl_root_table_list in
	 * ACPI driver.
	 *
	 * First step is surely finding RSDP via x86_default_get_root_pointer(), it is
	 * empty before, since team member's patch is merged, it can be got easily.
	 *
	 * Refer to get_rsdp_addr() for finding RSDP. Quick knowledge: for BIOS, RSDP exists
	 * in either first 1k of EBDA, or BIOS read-only memory space [0xE0000h, 0xFFFFF).
	 * It is worth to note: [0xE0000h, 0xFFFFF) belongs to BIOS ROM, SMBIOS may also be
	 * here. For me, it also implies that these tables are prepared beforehand, bundled
	 * with firmware, flashed into BIOS ROM together.
	 * (Might be memory shadowed into RAM?)
	 *
	 * If RSDP can't be obtained by shortcut, ACPI driver will search it manually via
	 * acpi_find_root_pointer(), which involves fixmap's __early_ioremap().
	 *
	 * After getting RSDP, parse tables via acpi_tb_parse_root_table().
	 */
	status = acpi_initialize_tables(initial_tables, ACPI_MAX_TABLES, 0);
	if (ACPI_FAILURE(status))
		return -EINVAL;

	/* ACPI tables from initrd is subsumed into ACPI driver's management. */
	acpi_table_initrd_scan();

	/* 吐槽：不得不说，acpi 的代码写的太太繁琐了！想找到变量赋值的地方要绕很久！*/

	/* ACPI employs concept of validate/invalidate table to manage table reference.
	 * Simply: 访问 table 时需 page mapping with fixmap range, then table reference
	 * count++; 访问结束则 count--. count = 0 means table is not used, then unmap.
	 * 好像这里是 early use ACPI, 所以要释放？以后会将 ACPI table 映射到永久使用区域？
	 *
	 * Table's virtual address is recorded in acpi_table_desc.pointer, which is in
	 * fixmap area IIRC.
	 */

	/* acpi_apic_instance is kernel parameter, default to 0.
	 * Quick reference:
	 *     2: use 2nd APIC table, if available; 1,0: use 1st APIC table.
	 */
	check_multiple_madt();
	return 0;
}
```
## early_acpi_boot_init

2nd ACPI initialization during boot process. Mainly for get APIC register base address from MADT. According to Intel SDM 3a, 10.4.1, APIC register initial starting address defaults to APIC_DEFAULT_PHYS_BASE, but Pentium 4, Intel Xeon, and P6 family processors permit relocating its starting address by modifying the value in the base address field of the IA32_APIC_BASE MSR.

```c
int __init early_acpi_boot_init(void)
{
	/* If acpi_disabled, bail out */
	if (acpi_disabled)
		return 1;

	/* Process the Multiple APIC Description Table (MADT), if present */
	early_acpi_process_madt();

	/* Hardware-reduced ACPI mode initialization: */
	/* 这个 indicator bit 在 FADT.flags: ACPI_FADT_HW_REDUCED. 上面 acpi_table_init
	 * 初始化内部 table array 时，已对 FADT 解析并初始化了 acpi_gbl_reduced_hardware.
	 * 看过 acpi_generic_reduced_hw_init 发现, HPET 是 ACPI hardware? */
	acpi_reduced_hw_init();

	return 0;
}

/* Self-documented name: record local APIC register base address from ACPI. */
#ifdef CONFIG_X86_LOCAL_APIC
static u64 acpi_lapic_addr __initdata = APIC_DEFAULT_PHYS_BASE;
#endif

/* Find APIC register base address from MADT. In case Linux kernel doesn't support
 * APIC, the function is null.
 *
 * But code logic is really confusing(to me), check my un-merged patch:
 *     https://lkml.org/lkml/2020/1/22/1500
 */
static void __init early_acpi_process_madt(void)
{
#ifdef CONFIG_X86_LOCAL_APIC
	int error;

	/*
	 * MADT header has "Local Interrupt Controller Address" field which records
	 * 32-bit physical base address for APIC register; MADT also has structure entry
	 * “Local APIC Address Override” who has 64-bit physical address overriding the
	 * APIC's base address;
	 *
	 * Only one "Local APIC Address Override Structure" maybe defined, if defined, its
	 * 64-bit address will be used for all local APICs.
	 *
	 * acpi_parse_madt() read 32-bit address from MADT header into acpi_lapic_addr.
	 *
	 * Complicated code, only register_lapic_address() worth to take a look.
	 */
	if (!acpi_table_parse(ACPI_SIG_MADT, acpi_parse_madt)) {

		/* Parse MADT LAPIC entries */
		/* "error" is confusing, it also indicates the number of processed
		 * structure entry.
		 */
		error = early_acpi_parse_madt_lapic_addr_ovr();
		if (!error) {
			acpi_lapic = 1; /* 说明 ACPI 中有正确的 lapic 信息? */
			/* find_smp_config -> xx -> smp_scan_config 已 set, if present. */
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

/* 入参是从 ACPI 中读出的 APIC base address */
void __init register_lapic_address(unsigned long address)
{
	mp_lapic_addr = address;

	/* x2apic_mode is initialized early in check_x2apic().
	 * Depending on having x2apic mode or not, 有2个情况：
	 *   - Not having x2apic mode: APIC registers are mapped into CPU's physical
	 *     address; then in code, they are accessed via virtual address. FIXMAP area
	 *     has reserved space for it: FIX_APIC_BASE.
	 *
	 *   - Having x2apic mode: APIC registers are MSR. They are accessed via instruction
	 *     wrmsr/rdmsr.
	 *
	 *  Refer: Intel SDM 3a, 10.12.1.2 x2APIC Register Address Space
	 */
	if (!x2apic_mode) {
		set_fixmap_nocache(FIX_APIC_BASE, address);
		apic_printk(APIC_VERBOSE, "mapped APIC to %16lx (%16lx)\n",
			    APIC_BASE, address);
	}

	/* The value are extracted directly from register, maybe that is what "physical" mean.
	 * Refer Intel SDM 3a, Figure 10-6. Local APIC ID Register & Figure 10-7. Local APIC
	 * Version Register for format.
	 *
	 * NOTE: there are 3 APIC ID source so far: initial APIC ID assigned by hardware
	 * during Power on; APIC ID from CPUID instruction; APIC ID from ACPI/MADT's "Processor
	 * Local APIC" structure entry. What's the difference/relation?
	 *
	 * Known facts: CPUID returns initial APIC ID; initial ID is written into APIC ID
	 * register, specific processor model allows software to modify this register.
	 * Refer to Intel SDM 3a, 10.4.6 Local APIC ID.
	 *
	 * Theoretically, all three APIC ID should be the same? What if not?
	 */
	if (boot_cpu_physical_apicid == -1U) {
		boot_cpu_physical_apicid  = read_apic_id();
		boot_cpu_apic_version = GET_APIC_VERSION(apic_read(APIC_LVR));
	}
}
```
## initmem_init

initmem_init() has different versions: 32-bit, 64-bit, NUMA, non-NUMA. Take (64-bit + NUMA)  for analysis. Refer to Documentation/vm/numa.rst for a general introduction of NUMA.

Tip: NUMA is acronym of Non-uniform memory access, which emphasize **memory**, then excerpt:

>For some architectures, such as x86, Linux will "hide" any node representing a physical cell that has no memory attached, and reassign any CPUs attached to that cell to a node representing a cell that does have memory. so when node doesn't have memory

start to make sense.

NUMA initialization is also analyzed in a [separate document](https://github.com/PinoTsao/Memo/blob/master/boot/numa_x86.mkd), but I forget had ever doing so ... And I find that until almost re-analyzed this function ... Please refer to it for extra info that is not in this document.

```c
void __init initmem_init(void)
{
	x86_numa_init();
}

/**
 * x86_numa_init - Initialize NUMA
 *
 * Try each configured NUMA initialization method until one succeeds.  The
 * last fallback is dummy single node config encompassing whole memory and
 * never fails.
 */
void __init x86_numa_init(void)
{
	if (!numa_off) {
#ifdef CONFIG_ACPI_NUMA
		/* Hardware NUMA topology information is from ACPI(SRAT & SLIT). */
		if (!numa_init(x86_acpi_numa_init))
			return;
#endif

#ifdef CONFIG_AMD_NUMA
	/* Old style AMD Opteron NUMA detection, SKIP */
#endif
	}

	numa_init(dummy_numa_init);
}

static int __init numa_init(int (*init_func)(void))
{
	int i;
	int ret;

	/* Redundant? Already initialized at when define. Probably a patch? */
	for (i = 0; i < MAX_LOCAL_APIC; i++)
		set_apicid_to_node(i, NUMA_NO_NODE);

	/* Tip: kernel max supported nodes number is defined by CONFIG_NODES_SHIFT. */
	nodes_clear(numa_nodes_parsed);
	nodes_clear(node_possible_map);
	nodes_clear(node_online_map);

	/* NR_NODE_MEMBLKS exist since Linux kernel imported into git. According to its
	 * X86 definition, seems only 2 memory regions on a node are allowed by Linux kernel?
	 * But E820_MAX_ENTRIES think it can has 3 range?
	 */
	memset(&numa_meminfo, 0, sizeof(numa_meminfo));

	/* Self-documented. */
	WARN_ON(memblock_set_node(0, ULLONG_MAX, &memblock.memory,
				  MAX_NUMNODES));
	WARN_ON(memblock_set_node(0, ULLONG_MAX, &memblock.reserved,
				  MAX_NUMNODES));

	/* In case that parsing SRAT failed. */
	/* Parsing SRAT failure means NUMA will not work, so does memory hotplug. */
	WARN_ON(memblock_clear_hotplug(0, ULLONG_MAX));

	/* Data of numa_distance[] derive from SLIT's matrix entry, denotes distance
	 * between node.
	 * Seems no necessary to reset? Anybody set it before?
	 */
	numa_reset_distance();

	/* ACPI's SRAT & SLIT initialization. */
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

	/* numa_meminfo is initialized during parsing ACPI_SRAT_TYPE_MEMORY_AFFINITY entry
	 * via acpi_numa_memory_affinity_init(), but sanitize until now? In initialization,
	 * only hotplug flags of raw NUMA data is used for marking it in memblock. The
	 * sanitization here only deal with memory range.
	 *
	 * Detail of sanitizing numa_meminfo:
	 *   1. move numa_memblk which is not in memblock.memory to numa_reserved_meminfo,
	 *      remove empty(start >= end) numa_memblk.  USAGE?
	 *   2. It can be inferred that Linux kernel regard ACPI_SRAT_TYPE_MEMORY_AFFINITY
	 *      structure's range to be canonical, which means the ranges are adjacent in
	 *      ascending order, whether overlapping or there is hole between ranges.
	 *   3. Overlapped ranges with different NID means the whole raw data is
	 *      unreliable, results in NUMA initialization failure.
	 *   4. Merging overlapped neighbor code is quite old & complex, skip.
	 */
	ret = numa_cleanup_meminfo(&numa_meminfo);
	if (ret < 0)
		return ret;

	/* Emulate NUMA node on platform that doesn't support NUMA.
	 * *ONLY FOR* debugging NUMA.   Details at below.
	 */
	numa_emulation(&numa_meminfo, numa_distance_cnt);

	/* Allocate memory for NODE_DATA & set node online. details at below. */
	ret = numa_register_memblks(&numa_meminfo);
	if (ret < 0)
		return ret;

	/* nr_cpu_ids is defined in smp.c as CONFIG_NR_CPUS, but could be updated by
	 * early parameter "nr_cpus=".
	 * Early PerCPU variable x86_cpu_to_node_map is initialized with NUMA_NO_NODE
	 * in definition, refer to chapter "Early PerCPU variable & PerCPU variable"
	 * in [boot link] for some insights of early PerCPU variable.
	 */
	for (i = 0; i < nr_cpu_ids; i++) {
		int nid = early_cpu_to_node(i);

		if (nid == NUMA_NO_NODE)
			continue;
		if (!node_online(nid))
			numa_clear_node(i);
	}

	/* Initialize x86_cpu_to_node_map from onlined node ID in round-robin way.
	 * Not get the idea behind even read the comments. Wait to see.
	 */
	numa_init_array();

	return 0;
}

int __init x86_acpi_numa_init(void)
{
	int ret;

	/* Parse SRAT & SLIT. Details as following. */
	ret = acpi_numa_init();
	if (ret < 0)
		return ret;
	return srat_disabled() ? -EINVAL : 0;
}

/*
 * Linux kernel defines logical node ID for internal use, which is mapped to
 * "proximity ID" defined by ACPI. Kernel's node ID *is* continuous.
 * Any chance that PXM ID is not continuous? According to SLIT, who has number of PXM
 * & distance matrix, PXM ID SHOULD be continuous from 0, or else it is buggy.
 */
/* drivers/acpi/numa.c */
int __init acpi_numa_init(void)
{
	int cnt = 0;

	if (acpi_disabled)
		return -EINVAL;

	/*
	 * Should not limit number with cpu num that is from NR_CPUS or nr_cpus=
	 * SRAT cpu entries could have different order with that in MADT.
	 * So go over all cpu entries in SRAT to get apicid to node mapping.
	 */
	/* Kernel 配置的 CPU 数目(config 或 parameter) 不应限制这里 SRAT 的解析 */

	/* SRAT: System Resource Affinity Table */
	/*
	 * SRAT handler acpi_parse_srat does nothing but get acpi_srat_revision.
	 * Leave all the work to following subtable handler.
	 *
	 * SRAT's sub-table has 2 CPU types & 1 memory type for x86. GICC(Generic Interrupt
	 * Controller CPU) is of ARM.  This is drivers/acpi, so it covers all architecture.
	 */
	if (!acpi_table_parse(ACPI_SIG_SRAT, acpi_parse_srat)) {
		struct acpi_subtable_proc srat_proc[3];

		 /*
		 * Take ACPI_SRAT_TYPE_CPU_AFFINITY for example, the sub-table has APIC ID &
		 * corresponding proximity domain ID.  PXM ID(32-bit) is defined by ACPI, its
		 * counterpart in kernel is NUMA node ID(MAX_NUMNODES), the handler maps them up
		 * via pxm_to_node_map[] & node_to_pxm_map[].
		 * Mapping strategy:
		 * For each PXM ID, first unset bit's index in nodes_found_map is taken as its
		 * node ID. So actually a PXM's node ID is determined by appearance sequence
		 * of ACPI_SRAT_TYPE_CPU_AFFINITY structure.
		 *
		 * Obviously, 32-bit PXM ID should not be great than kernel's MAX_NUMNODES,
		 * or else "node ID" won't enough for mapping "PXM ID", that is why:
		 *
		 *         #define MAX_PXM_DOMAINS MAX_NUMNODES
		 *
		 * If PXM ID does is great than MAX_NUMNODES, then it is omitted.
		 *
		 * After getting NUMA node ID，physical APIC ID is also mapped to it via
		 * __apicid_to_node[]. Usage of numa_nodes_parsed is tricky, it is also set
		 * when parsing memory affinity structure, considering node may have no memory,
		 * things start to look complex.
		 *
		 * NOTE: one node could have multiple logical CPUs, i.e., APIC ID.
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

		/*
		 * Memory Affinity Structure is handled separately, because node ID is needed,
		 * which is obtained during APIC structure processing.
		 *
		 * store range data into numa_meminfo. Set set numa_nodes_parsed again after
		 * parsing.	Re-evaluate max_possible_pfn after parsing each range.
		 */
		cnt = acpi_table_parse_srat(ACPI_SRAT_TYPE_MEMORY_AFFINITY,
					    acpi_parse_memory_affinity, 0);
	}

	/* After processing SRAT, now kernel is aware of APIC IDs & memory ranges
	 * that each node has. */

	/* SLIT: System Locality Information Table */
	/*
	 * Distance between nodes means memory latency, which is a one-byte value in ACPI.
	 * Value 0xFF means unreachable between two nodes; value 10 is the distance of a
	 * node to itself; valid distance is [10 - 255); distance values [0-9] are reserved
	 * and have no meaning.
	 *
	 * slit_valid() verify SLIT matrix data to make sure it is valid in following way:
	 *     1. distance of node to itself must be LOCAL_DISTANCE(10);
	 *     2. distance between nodes must be great than LOCAL_DISTANCE(10);
	 *
	 * It can be seen from acpi_numa_slit_init() that PXM ID is considered continuous.
	 *
	 * SLIT parsing is to initialize numa_distance[] from matrix entry, and record
	 * node count into numa_distance_cnt.
	 *
	 * 有必要再次分析 numa_alloc_distance, 因为产生了和 474aeffd88b8 一样的想法，但它
	 * 又被 b678c91aefa7 revert. Details at: https://lkml.org/lkml/2017/3/13/1231
	 */
	acpi_table_parse(ACPI_SIG_SLIT, acpi_parse_slit);

	if (cnt < 0)
		return cnt;
	else if (!parsed_numa_memblks)
		return -ENOENT;
	return 0;
}
```

numa_register_memblks() aims for allocate NODE_DATA.

>Tips:
**memblock.memory** is initialized from E820 info but with only range info & is not aware of NUMA info, i.e., memory range <-> nid,  memory hot pluggable property. Thus certain memblock region may have both hot-pluggable & non-hot-pluggable part.

```c
static int __init numa_register_memblks(struct numa_meminfo *mi)
{
	int i, nid;

	/* First initialization of node_possible_map.
	 * Perplexity as 474aeffd88b8 told still exist ...
	 */
	/* Account for nodes with cpus and no memory */
	node_possible_map = numa_nodes_parsed;
	numa_nodemask_from_meminfo(&node_possible_map, mi);
	if (WARN_ON(nodes_empty(node_possible_map)))
		return -EINVAL;

	for (i = 0; i < mi->nr_blks; i++) {
		struct numa_memblk *mb = &mi->blk[i];
		/* Must read its comments. */
		memblock_set_node(mb->start, mb->end - mb->start,
				  &memblock.memory, mb->nid);
	}

	/* Quite straightforward code & comments, no need for verbiage. */
	/*
	 * At very early time, the kernel have to use some memory such as
	 * loading the kernel image. We cannot prevent this anyway. So any
	 * node the kernel resides in should be un-hotpluggable.
	 *
	 * And when we come here, alloc node data won't fail.
	 */
	numa_clear_kernel_node_hotplug();

	/* Need more background knowledge, get back later. */
	/*
	 * If sections array is gonna be used for pfn -> nid mapping, check
	 * whether its granularity is fine enough.
	 */
	if (IS_ENABLED(NODE_NOT_IN_PAGE_FLAGS)) {
		unsigned long pfn_align = node_map_pfn_alignment();

		if (pfn_align && pfn_align < PAGES_PER_SECTION) {
			pr_warn("Node alignment %LuMB < min %LuMB, rejecting NUMA config\n",
				PFN_PHYS(pfn_align) >> 20,
				PFN_PHYS(PAGES_PER_SECTION) >> 20);
			return -EINVAL;
		}
	}

	/* Just do the sanity check.
	 *
	 * Tip: both ranges in memblock.memroy & numa_meminfo are sanitized.
	 * memblock.memory has *all* available RAM range info for kernel which is from e820,
	 * memory allocation doesn't affect the range info inside.
	 */
	if (!numa_meminfo_cover_memory(mi))
		return -EINVAL;

	/* Finally register nodes. */
	for_each_node_mask(nid, node_possible_map) {
		u64 start = PFN_PHYS(max_pfn);
		u64 end = 0;

		/* Get the start & end address from all ranges of a node for the reason stated
		 * in the comments below. Hole?
		 */
		for (i = 0; i < mi->nr_blks; i++) {
			if (nid != mi->blk[i].nid)
				continue;
			start = min(mi->blk[i].start, start);
			end = max(mi->blk[i].end, end);
		}

		if (start >= end)
			continue;

		/*
		 * Don't confuse VM with a node that doesn't have the
		 * minimum amount of memory:
		 */
		if (end && (end - start) < NODE_MIN_SIZE)
			continue;

		/* NODE_DATA is allocated as locally as possible, i.e., in the node it represents.
		 * Just allocate the memory.  Refer to bd5cfb8977fbb49 for extra info.
		 * NOTE: Node has NODE_DATA allocated is set to online state right away.
		 */
		alloc_node_data(nid);
	}

	/* Dump memblock with node info and return. */
	memblock_dump_all();
	return 0;
}

/*
 * Sanity check to catch more bad NUMA configurations (they are amazingly
 * common).  Make sure the nodes cover all memory.
 */
static bool __init numa_meminfo_cover_memory(const struct numa_meminfo *mi)
{
	u64 numaram, e820ram;
	int i;

	/* Linux kernel regard the range info in memblock.memory is reliable for the reason
	 * stated in comments above. Check memory range info from NUMA against it to get
	 * real RAM size from NUMA perspective by trimming memory hole ranges.
	 */
	numaram = 0;
	for (i = 0; i < mi->nr_blks; i++) {
		u64 s = mi->blk[i].start >> PAGE_SHIFT;
		u64 e = mi->blk[i].end >> PAGE_SHIFT;
		numaram += e - s;
		numaram -= __absent_pages_in_range(mi->blk[i].nid, s, e);
		if ((s64)numaram < 0)
			numaram = 0;
	}

	/* Get real RAM size info from e820 perspective. */
	e820ram = max_pfn - absent_pages_in_range(0, max_pfn);

	/* If the delta is more than 1M, Linux kernel regard ACPI NUMA memory info is
	 * un-reliable, fail the initialization and fall back to dummy_numa_init().
	 */
	/* We seem to lose 3 pages somewhere. Allow 1M of slack. */
	if ((s64)(e820ram - numaram) >= (1 << (20 - PAGE_SHIFT))) {
		printk(KERN_ERR "NUMA: nodes only cover %LuMB of your %LuMB e820 RAM. Not used.\n",
		       (numaram << PAGE_SHIFT) >> 20,
		       (e820ram << PAGE_SHIFT) >> 20);
		return false;
	}
	return true;
}
```

It worth to take a look when ACPI NUMA initialization fail and fall back to dummy_numa_init:

```c
/**
 * dummy_numa_init - Fallback dummy NUMA init
 *
 * Used if there's no underlying NUMA architecture, NUMA initialization
 * fails, or NUMA is disabled on the command line.
 *
 * Must online at least one node and add memory blocks that cover all
 * allowed memory.  This function must not fail.
 */
static int __init dummy_numa_init(void)
{
	printk(KERN_INFO "%s\n",
	       numa_off ? "NUMA turned off" : "No NUMA configuration found");
	printk(KERN_INFO "Faking a node at [mem %#018Lx-%#018Lx]\n",
	       0LLU, PFN_PHYS(max_pfn) - 1);

	node_set(0, numa_nodes_parsed);
	numa_add_memblk(0, 0, PFN_PHYS(max_pfn));

	return 0;
}
```

## x86_64's paging_init

Defined in *init_64.c*:

```c
void __init paging_init(void)
{
	sparse_init();

	/*
	 * clear the default setting with node 0
	 * note: don't use nodes_clear here, that is really clearing when
	 *	 numa support is not compiled in, and later node_set_state
	 *	 will not set it back.
	 */
	node_clear_state(0, N_MEMORY);
	node_clear_state(0, N_NORMAL_MEMORY);

	zone_sizes_init();
}
```

It involves initialization of sparse memory model & concept of memory zone. Refer to *Documentation/vm/memory-model.rst* for introduction of sparse memory model, which is a must-read before reading the code.

Following analysis will base on CONFIG_SPARSEMEM_EXTREME & CONFIG_SPARSEMEM_VMEMMAP. Section size is 128Mb under x86_64.

```c
/*
 * Allocate the accumulated non-linear sections, allocate a mem_map
 * for each and record the physical to section mapping.
 */
void __init sparse_init(void)
{
	unsigned long pnum_end, pnum_begin, map_count = 1;
	int nid_begin;

	/* Iterate through all RAM ranges, allocate & initialize all mem_section
	 * structure.  Details analyzed below. */
	memblocks_present();

	/* "p" might means "present". */
	pnum_begin = first_present_section_nr();
	nid_begin = sparse_early_nid(__nr_to_section(pnum_begin));

	/* Setup pageblock_order for HUGETLB_PAGE_SIZE_VARIABLE */
	set_pageblock_order();

	for_each_present_section_nr(pnum_begin + 1, pnum_end) {
		int nid = sparse_early_nid(__nr_to_section(pnum_end));

		if (nid == nid_begin) {
			map_count++;
			continue;
		}
		/* Init node with sections in range [pnum_begin, pnum_end) */
		/* Section range [pnum_begin, pnum_end -1] belongs to the same node. */
		sparse_init_nid(nid_begin, pnum_begin, pnum_end, map_count);
		nid_begin = nid;
		pnum_begin = pnum_end;
		map_count = 1;
	}

	sparse_init_nid(nid_begin, pnum_begin, pnum_end, map_count);
	vmemmap_populate_print_last();
}

static void __init memblocks_present(void)
{
	unsigned long start, end;
	int i, nid;

	for_each_mem_pfn_range(i, MAX_NUMNODES, &start, &end, &nid)
		memory_present(nid, start, end);
}

#ifdef CONFIG_SPARSEMEM_EXTREME
    #define SECTIONS_PER_ROOT       (PAGE_SIZE / sizeof (struct mem_section))
#else
    /* Actually one-dimensional array in vertical way. */
    #define SECTIONS_PER_ROOT       1
#endif

#define SECTION_NR_TO_ROOT(sec) ((sec) / SECTIONS_PER_ROOT)
#define NR_SECTION_ROOTS        DIV_ROUND_UP(NR_MEM_SECTIONS, SECTIONS_PER_ROOT)
#define SECTION_ROOT_MASK       (SECTIONS_PER_ROOT - 1)

#ifdef CONFIG_SPARSEMEM_EXTREME
    struct mem_section **mem_section;
#else
    struct mem_section mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT]
            ____cacheline_internodealigned_in_smp;
#endif

/* Record a memory area against a node. */
static void __init memory_present(int nid, unsigned long start, unsigned long end)
{
	unsigned long pfn;

#ifdef CONFIG_SPARSEMEM_EXTREME
	/* Allocate memory for the row of 2-dimensional array of mem_section.  As
	 * it is only allocated once, that is how "unlikely()" make sense.
	 */
	if (unlikely(!mem_section)) {
		unsigned long size, align;

		size = sizeof(struct mem_section *) * NR_SECTION_ROOTS;
		align = 1 << (INTERNODE_CACHE_SHIFT);
		mem_section = memblock_alloc(size, align);
		if (!mem_section)
			panic("%s: Failed to allocate %lu bytes align=0x%lx\n",
			      __func__, size, align);
	}
#endif

	/* section number, also is the 1st PFN of current section. Ideally, start PFN of
	 * range should be at the start boundary of a section.
	 */
	start &= PAGE_SECTION_MASK;
	mminit_validate_memmodel_limits(&start, &end);

	/* "start" is forced at the start boundary of section, but what if "end" is
	 * not at the end boundary? Assume range size is always multitude of section
	 * size(128Mb) for now, then there will be no lost pages.
	 */
	for (pfn = start; pfn < end; pfn += PAGES_PER_SECTION) {
		unsigned long section = pfn_to_section_nr(pfn);
		struct mem_section *ms;

		/* Find column index according to section number in 2-dimensional array,
		 * allocate for whole column in current node.
		 * 找到 section 所在 column(从上到下，从走到右), 在当前 node 分配该 column 表示
		 * 的所有 mem_section.
		 */
		sparse_index_init(section, nid);
		/* Map section number to its node ID. */
		set_section_nid(section, nid);

			/* Get the exact struct mem_section according to section number. */
		ms = __nr_to_section(section);
		if (!ms->section_mem_map) {
			ms->section_mem_map = sparse_encode_early_nid(nid) |
							SECTION_IS_ONLINE;
			/* 二维数组中，section number 是从上到下，从左到右 数。*/
			section_mark_present(ms);
		}
	}
}

/*
 * Initialize sparse on a specific node. The node spans [pnum_begin, pnum_end)
 * And number of present sections in this node is map_count.
 */
static void __init sparse_init_nid(int nid, unsigned long pnum_begin,
				   unsigned long pnum_end,
				   unsigned long map_count)
{
	/* Section is divided into 2Mb sub-sections. */
	struct mem_section_usage *usage;
	unsigned long pnum;
	struct page *map;

	usage = sparse_early_usemaps_alloc_pgdat_section(NODE_DATA(nid),
			mem_section_usage_size() * map_count);
	if (!usage) {
		pr_err("%s: node[%d] usemap allocation failed", __func__, nid);
		goto failed;
	}
	sparse_buffer_init(map_count * section_map_size(), nid);
	for_each_present_section_nr(pnum_begin, pnum) {
		unsigned long pfn = section_nr_to_pfn(pnum);

		if (pnum >= pnum_end)
			break;

		map = __populate_section_memmap(pfn, PAGES_PER_SECTION,
				nid, NULL);
		if (!map) {
			pr_err("%s: node[%d] memory map backing failed. Some memory will not be available.",
			       __func__, nid);
			pnum_begin = pnum;
			sparse_buffer_fini();
			goto failed;
		}
		check_usemap_section_nr(nid, usage);
		sparse_init_one_section(__nr_to_section(pnum), pnum, map, usage,
				SECTION_IS_EARLY);
		usage = (void *) usage + mem_section_usage_size();
	}
	sparse_buffer_fini();
	return;
failed:
	/* We failed to allocate, mark all the following pnums as not present */
	for_each_present_section_nr(pnum_begin, pnum) {
		struct mem_section *ms;

		if (pnum >= pnum_end)
			break;
		ms = __nr_to_section(pnum);
		ms->section_mem_map = 0;
	}
}
```


## acpi_boot_init

前两个 acpi boot 初始化函数和这个函数距离有点远，我们一次性把 ACPI 初始化分析完。

```c
int __init acpi_boot_init(void)
{
	/* those are executed after early-quirks are executed */
	dmi_check_system(acpi_dmi_table_late);

	/* If acpi_disabled, bail out	 */
	if (acpi_disabled)
		return 1;

	/* Duplicate. Check it out:
	 * https://lore.kernel.org/linux-pm/24266640.LfmLNjZWAc@kreacher
	 */
	acpi_table_parse(ACPI_SIG_BOOT, acpi_parse_sbf);

	/* set sci_int and PM timer address. FADT 在 acpi_boot_table_init - x - x -
	 * -> acpi_tb_parse_fadt 已解析过，且将内容 copy 到 local 变量 acpi_gbl_FADT,
	 * 并做了格式转换。 acpi_parse_fadt 中只是使用 acpi_gbl_FADT.
	 * NOTE: FADT 针对 ARM 和 IA-PC 分别定义了 2-byte long flag(ARM_BOOT_ARCH,
	 * IAPC_BOOT_ARCH); PM timer 位于 I/O address space. */
	acpi_table_parse(ACPI_SIG_FADT, acpi_parse_fadt);

	/* Process the Multiple APIC Description Table (MADT), if present */
	/* 涉及太多中断相关的知识，待有时间详细分析，目前只有简要分析。*/
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

/* 之前已 early_acpi_process_madt. */
static void __init acpi_process_madt(void)
{
#ifdef CONFIG_X86_LOCAL_APIC
	int error;

	/* acpi_parse_madt 中的干货仅是 MADT header 初始化 acpi_lapic_addr. Early
	 * process MADT 时还会额外解析 Local APIC Address Override Structure, 若有此
	 * structure, 这里 acpi_lapic_addr 岂不是可能再次 override? 很快找到答案：该变量只
	 * 暂存 ACPI 读来的地址，最终通过 register_lapic_address() 记录在 mp_lapic_addr. */
	if (!acpi_table_parse(ACPI_SIG_MADT, acpi_parse_madt)) {
		/* Parse MADT LAPIC entries. 见下面详细分析 */
		error = acpi_parse_madt_lapic_entries();
		if (!error) {
			/* early process 时已 set, 这里 set 应是无 override structure 的情况。*/
			acpi_lapic = 1;

			/* Parse MADT IO-APIC entries	 */
			/* 细节在下文。*/
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
	if (acpi_lapic && acpi_ioapic)
		printk(KERN_INFO "Using ACPI (MADT) for SMP configuration "
		       "information\n");
	else if (acpi_lapic)
		printk(KERN_INFO "Using ACPI for processor (LAPIC) "
		       "configuration information\n");
#endif
	return;
}

/* parse ACPI 中 APIC 相关信息 */
static int __init acpi_parse_madt_lapic_entries(void)
{
	int count;
	int x2count = 0;
	int ret;
	struct acpi_subtable_proc madt_proc[2];

	if (!boot_cpu_has(X86_FEATURE_APIC))
		return -ENODEV;

	/* Parse ACPI_MADT_TYPE_LOCAL_SAPIC first?
	 * SAPIC = Streamlined APIC: An advanced APIC commonly found on Intel
	 * Itanium TM Processor Family-based 64-bit systems. Soga, SAPIC 和
	 * APIC/X2APIC 不会同时存在，因为他们是不同 CPU arch 上的 APIC. So, if NO SAPIC,
	 * check APIC/X2APIC, 逻辑没问题。
	 *
	 * SAPIC sub-table 中有多个 id field, 初看很困惑。EID 是 Extension ID 的缩写,
	 * ID & EID 一起描述 SAPIC 的 APIC ID, 这是硬件拓扑范畴的 ID; 同时, SAPIC 在
	 * ACPI namespace 还有 ACPI processor ID 和 ACPI Processor UID Value, 表示某
	 * CPU 在 ACPI namespace 中的 ID, 看起来应该只用其中之一，而且前者在 ACPI Spec
	 * 中标记为 deprecated. 但代码处理中, SAPIC 与 APIC/X2APIC 不同，使用了前者，APIC
	 * & X2APIC structure 中根本都没有前者 ID.
	 *
	 * (APIC ID & APIC's ACPI ID) 经 acpi_register_lapic -> generic_processor_info
	 * 映射得到 kernel 视角的 logical CPU number. 细节见下文。
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

/* 简约分析 to get the point. */
int generic_processor_info(int apicid, int version)
{
	/* nr_cpu_ids 初始化自 CONFIG_NR_CPUS, 也可被修改 via parameter "nr_cpus".
	 * 表示 kernel 支持的最大 CPU 数。 */
	int cpu, max = nr_cpu_ids;

	/* boot_cpu_physical_apicid (之前)在 early_acpi_parse_madt_lapic_addr_ovr --
	 * --> register_lapic_address 已由 APIC ID register 初始化。在本文分析条件下
	 * (X86_64 & SMP), it is not set in phys_cpu_present_map.
	 * (之后的start_kernel 的 time_init 在 UniProcessor 情况下会 set.) */
	bool boot_cpu_detected = physid_isset(boot_cpu_physical_apicid,
				phys_cpu_present_map);

	/* Skip 无关紧要 codes. */

	/* If boot cpu has not been detected yet, then only allow up to
	 * nr_cpu_ids - 1 processors and keep one slot free for boot cpu.
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
	/* kernel parameter "noapic" 指的是不要使用 IO-APIC, 但这个名字也太名不副实了？*/
	if (skip_ioapic_setup) {
		printk(KERN_INFO PREFIX "Skipping IOAPIC probe "
		       "due to 'noapic' option.\n");
		return -ENODEV;
	}

	/* Quick knowledge: 系统中可能有一或多个 I/O APIC, each I/O APIC resides at a
	 * unique address, address 指的应是 direct register 中的 index register 的
	 * memory address. 每个 I/O APIC chip 对应 ACPI MADT 中一个 I/O APIC Structure.
	 *
	 * IO-APIC structure 的处理也很复杂，在 mp_register_ioapic. Kernel 用 struct
	 * ioapic 描述一个 I/O APIC, 用 nr_ioapics 表示它的数目，该函数的主要作用是初始化
	 * 相应的 struct ioapic(ioapics[MAX_IO_APICS]).
	 *
	 * 术语的不同：kernel 用 struct ioapic.nr_registers 描述 I/O APIC's redirection
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

