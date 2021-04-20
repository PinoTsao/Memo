# memblock

An overview introduction can be found at the head comments of mm/memblock.c.  Here we analyse some usual memblock routines that are frequently seen in the mainline analysis to get an idea about its framework.

### memblock_add_range

“memory” & “reserved” regions are added through it in ascending way. It handles with physical address. Remember: **the region is added in ascending way, no overlapping between added regions**, that will help to understand the code.

```c
static int __init_memblock memblock_add_range(struct memblock_type *type,
				phys_addr_t base, phys_addr_t size,
				int nid, enum memblock_flags flags)
{
	bool insert = false;
	phys_addr_t obase = base; /* o for original */
	phys_addr_t end = base + memblock_cap_size(base, &size);
	int idx, nr_new;
	struct memblock_region *rgn;

	if (!size)
		return 0;

	/* The 1st region is a special one. Add it straightforward. */
	/* special case for empty array */
	if (type->regions[0].size == 0) {
		WARN_ON(type->cnt != 1 || type->total_size);
		type->regions[0].base = base;
		type->regions[0].size = size;
		type->regions[0].flags = flags;
		memblock_set_region_node(&type->regions[0], nid);
		type->total_size = size;
		return 0;
	}
repeat:
	/*
	 * The following is executed twice.  Once with %false @insert and
	 * then with %true.  The first counts the number of regions needed
	 * to accommodate the new area.  The second actually inserts them.
	 */
	/* 上面注释解释的很清楚。*/
	base = obase;
	nr_new = 0;

	/* Iterate to get each added region, compare with new region. */
	for_each_memblock_type(idx, type, rgn) {
		phys_addr_t rbase = rgn->base; /* r for region. */
		phys_addr_t rend = rbase + rgn->size;

		/* New region is lower than "rgn", then break out to add it just before "rgn". */
		if (rbase >= end)
			break;
		/* New region is higher than current one, continue to see if it is still
		 * larger than the next one. */
		if (rend <= base)
			continue;
		/*
		 * @rgn overlaps.  If it separates the lower part of new
		 * area, insert that portion.
		 */
		/* When being here, suggests the 2 regions overlap, which has several conditions:
		 * all covered by following code:
		 *   1. base < rbase, end < rend.
		 *   2. base < rbase, end > rend, i.e., new region completely cover the current one.
		 *   3. base > rbase, end > rend, the opposite overlap versus 1.
		 */

		if (rbase > base) {
#ifdef CONFIG_NEED_MULTIPLE_NODES
			WARN_ON(nid != memblock_get_region_node(rgn));
#endif
			WARN_ON(flags != rgn->flags);
			nr_new++;
			if (insert)
				memblock_insert_region(type, idx++, base,
						       rbase - base, nid,
						       flags);
		}
		/* area below @rend is dealt with, forget about it */
		base = min(rend, end);
	}

	/* insert the remaining portion */
	/* Condition 2, which will add 2 regions. */
	if (base < end) {
		nr_new++;
		if (insert)
			memblock_insert_region(type, idx, base, end - base,
					       nid, flags);
	}

	if (!nr_new)
		return 0;

	/*
	 * If this was the first round, resize array and repeat for actual
	 * insertions; otherwise, merge and return.
	 */
	if (!insert) {
		while (type->cnt + nr_new > type->max)
			if (memblock_double_array(type, obase, size) < 0)
				return -ENOMEM;
		insert = true;
		goto repeat;
	} else {
		/* iterate all to merge. */
		memblock_merge_regions(type);
		return 0;
	}
}
```


### memblock_alloc_range_nid

All allocation finally go to memblock_alloc_range_nid() --> memblock_find_in_range_node() --> __memblock_find_range_bottom_up() or __memblock_find_range_top_down() --> for_each_free_mem_range() --..--> __next_mem_range().

Traditionally, memblock only allocate memory in top-down way, as NUMA emerged, bottom-up allocation is implemented.

The first memblock allocation takes place in e820__memblock_alloc_reserved_mpc_new() which has range limitation of 1M.

Just pick up the interesting points of each function for analysis.

**memblock_alloc_range_nid()**: memory region attributes MEMBLOCK_MIRROR is only available under EFI; the allocated region will be added to "reserved".

**memblock_find_in_range_node()**: start address will be lifted above 1st 4K if it is lower than that. But 1st page is already reserved at the beginning of setup_arch(), duplicate?

**__memblock_find_range_bottom_up()**: **clamp(val, lo, hi)** returns a value that between its *hi* & *lo*, 3 cases:

  1. *val* < *lo*, returns *lo*
  2. *lo < *val < *hi*, returns *val*
  3. *val* > *hi*, returns hi

**__next_mem_range()**: the core function to iterate all available memory region in bottom-up way. For each "memory" region, iterate **the deduced "memory" regions** from "reserved", compare them conservatively to get the real available range. What does **deduced** mean? Theoretically, a memory region should be type either "memory" or "reserved", but regions are initialized manually by calling **memblock_add** & **memblock_reserve**, so there is chance that a region is not added in both types. As added regions do not overlap, memblock could take the range between two "reserved" regions as "memory", then compare the deduced "memory" region with real "memory" region conservatively to get the real available range.