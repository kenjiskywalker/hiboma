## SReclaimable, SUnreclaim

```c
		"SReclaimable:   %8lu kB\n"
		"SUnreclaim:     %8lu kB\n"

// ...

		K(global_page_state(NR_SLAB_RECLAIMABLE) +
				global_page_state(NR_SLAB_UNRECLAIMABLE)),
```

__vm_enough_memory で参照されている
        
```c
		/*
		 * Any slabs which are created with the
		 * SLAB_RECLAIM_ACCOUNT flag claim to have contents
		 * which are reclaimable, under pressure.  The dentry
		 * cache and most inode caches should fall into this
		 */
		free += global_page_state(NR_SLAB_RECLAIMABLE);
```

slab (kmem_cache等) で reclaim できるもの/できないもののサイズを指している

## kmem_cache

SLAB_RECLAIM_ACCOUNT の有無で分類される

```
struct kmem_cache {
	unsigned int size, align;
	unsigned long flags;
	const char *name;
	void (*ctor)(void *);
};
```

kmem_cache_create に渡すフラグで変更できる様子

## NR_SLAB_RECLAIMABLE, NR_SLAB_UNRECLAIMABLE の減算

kmem_cache_shrink -> discard_slab -> free_slab -> **__free_slab**

```c
static void __free_slab(struct kmem_cache *s, struct page *page)
{
	int order = compound_order(page);
	int pages = 1 << order;

	if (unlikely(SLABDEBUG && PageSlubDebug(page))) {
		void *p;

		slab_pad_check(s, page);
		for_each_object(p, s, page_address(page),
						page->objects)
			check_object(s, page, p, 0);
		__ClearPageSlubDebug(page);
	}

	kmemcheck_free_shadow(page, compound_order(page));

	mod_zone_page_state(page_zone(page),
		(s->flags & SLAB_RECLAIM_ACCOUNT) ?
		NR_SLAB_RECLAIMABLE : NR_SLAB_UNRECLAIMABLE,
		-pages);

	__ClearPageSlab(page);
	reset_page_mapcount(page);
	if (current->reclaim_state)
		current->reclaim_state->reclaimed_slab += pages;
	__free_pages(page, order);
}
```

## NR_SLAB_RECLAIMABLE, NR_SLAB_UNRECLAIMABLE の加算

kmem_cache_alloc -> slab_alloc __slab_alloc -> new_slab -> **allocate_slab**

```c
static struct page *allocate_slab(struct kmem_cache *s, gfp_t flags, int node)
{
	struct page *page;
	struct kmem_cache_order_objects oo = s->oo;
	gfp_t alloc_gfp;

	flags |= s->allocflags;

	/*
	 * Let the initial higher-order allocation fail under memory pressure
	 * so we fall-back to the minimum order allocation.
	 */
	alloc_gfp = (flags | __GFP_NOWARN | __GFP_NORETRY) & ~__GFP_NOFAIL;

	page = alloc_slab_page(alloc_gfp, node, oo);
	if (unlikely(!page)) {
		oo = s->min;
		/*
		 * Allocation may have failed due to fragmentation.
		 * Try a lower order alloc if possible
		 */
		page = alloc_slab_page(flags, node, oo);
		if (!page)
			return NULL;

		stat(get_cpu_slab(s, raw_smp_processor_id()), ORDER_FALLBACK);
	}

	if (kmemcheck_enabled
		&& !(s->flags & (SLAB_NOTRACK | DEBUG_DEFAULT_FLAGS))) {
		int pages = 1 << oo_order(oo);

		kmemcheck_alloc_shadow(page, oo_order(oo), flags, node);

		/*
		 * Objects from caches that have a constructor don't get
		 * cleared when they're allocated, so we need to do it here.
		 */
		if (s->ctor)
			kmemcheck_mark_uninitialized_pages(page, pages);
		else
			kmemcheck_mark_unallocated_pages(page, pages);
	}

	page->objects = oo_objects(oo);
	mod_zone_page_state(page_zone(page),
		(s->flags & SLAB_RECLAIM_ACCOUNT) ?
		NR_SLAB_RECLAIMABLE : NR_SLAB_UNRECLAIMABLE,
		1 << oo_order(oo));

	return page;
}
```