# LRU, inactive_list, active_list

![2014-06-06 17 24 00](https://cloud.githubusercontent.com/assets/172456/3198095/1d04d80c-ed54-11e3-813a-b1754d92e552.png)

[Professional Linux Kernel Architecture](http://www.amazon.com/dp/0470343435)

----

 * 一応 split LRU 以前の図なので注意
 * 1 node の 1 zone を指してる図

## アルゴリズム 

 * LRU
 * second chance 

## 主要な関数群

 * lru_cache_add, lru_cache_add_active
   * split LRU では lru_cache_add_file, lru_cache_add_anon, lru_cache_add_active_anon, lru_cache_add_active_file に分離
   * いずれも __lru_cache_add を呼び出すので実装は一緒。対象とする LRU が違うだけ。
 * mark_page_accessed
 * page_referenced
   * shrink_page_list -> page_check_references -> ... で呼び出しされる
   * 名前と裏腹に副作用があるのだ
 * shrink_active_list
   * shrink_inactive_list も大事
 * activate_page
 * SetPageActive
 * ClearPageActive

### lru_cache_add

split LRU では lru_cache_add_file, lru_cache_add_anon が を呼び出す

__lru_cache_add

  * pagevec に繋ぐ
    * get_cpu_var でスピンロックのいらないリスト
  * pagevec が一杯なら __pagevec_lru_add で LRUリストに繋ぐ
  * _active 属も一緒の実装を使う

```c
 void __lru_cache_add(struct page *page, enum lru_list lru)
{
	struct pagevec *pvec = &get_cpu_var(lru_add_pvecs)[lru];

	page_cache_get(page);
	if (!pagevec_add(pvec, page))
		____pagevec_lru_add(pvec, lru);
	put_cpu_var(lru_add_pvecs);
}
EXPORT_SYMBOL(__lru_cache_add);
```

____pagevec_lru_add

 * pagevec -> LRUリストに繋ぐ処理

```c
/*
 * Add the passed pages to the LRU, then drop the caller's refcount
 * on them.  Reinitialises the caller's pagevec.
 */
void ____pagevec_lru_add(struct pagevec *pvec, enum lru_list lru)
{
	int i;
	struct zone *zone = NULL;

	VM_BUG_ON(is_unevictable_lru(lru));

    // pagevec (vector = 配列) をイテレート
	for (i = 0; i < pagevec_count(pvec); i++) {

        // pagevec は zone の区別無くキャッシュしている
		struct page *page = pvec->pages[i];
		struct zone *pagezone = page_zone(page);
		int file;
		int active;

        // 扱っている zone が違う。別のゾーンに繋ぐ
		if (pagezone != zone) {
			if (zone)
				spin_unlock_irq(&zone->lru_lock);
			zone = pagezone;
			spin_lock_irq(&zone->lru_lock);
		}
		VM_BUG_ON(PageActive(page));
		VM_BUG_ON(PageUnevictable(page));
		VM_BUG_ON(PageLRU(page));

        // LRU に載っているフラグ?
		SetPageLRU(page);
		active = is_active_lru(lru);
		file = is_file_lru(lru);

        // Active なページのフラグ
		if (active)
			SetPageActive(page);
		update_page_reclaim_stat(zone, page, file, active);

        // ここで LRUリストに繋ぐ
		add_page_to_lru_list(zone, page, lru);
	}
	if (zone)
		spin_unlock_irq(&zone->lru_lock);

    // pagevec の page を消す?
	release_pages(pvec->pages, pvec->nr, pvec->cold);
	pagevec_reinit(pvec);
}

EXPORT_SYMBOL(____pagevec_lru_add);
```

### lru_cache_add

### mark_page_accessed

```c 
/*
 * Mark a page as having seen activity.
 *
 * inactive,unreferenced	->	inactive,referenced   // B
 * inactive,referenced		->	active,unreferenced   // A
 * active,unreferenced		->	active,referenced     // B
 */
void mark_page_accessed(struct page *page)
{
	if (!PageActive(page) && !PageUnevictable(page) &&
			PageReferenced(page) && PageLRU(page)) {
        // パターンA active リストに繋いで referenced を落とす
		activate_page(page);
		ClearPageReferenced(page);
	} else if (!PageReferenced(page)) {
        // パターンB referenced をつけるだけ
		SetPageReferenced(page);
	}
}

EXPORT_SYMBOL(mark_page_accessed);
```

### page_referenced

 * KSM, anon, file によって分岐する
 * 名前からは分からないけど referenced ビットを落とす副作用持っているので注意
   * test_and_clear_referenced でビットを落とす

```c
/**
 * page_referenced - test if the page was referenced
 * @page: the page to test
 * @is_locked: caller holds lock on the page
 * @mem_cont: target memory controller
 * @vm_flags: collect encountered vma->vm_flags who actually referenced the page
 *
 * Quick test_and_clear_referenced for all mappings to a page,
 * returns the number of ptes which referenced the page.
 */
int page_referenced(struct page *page,
		    int is_locked,
		    struct mem_cgroup *mem_cont,
		    unsigned long *vm_flags)
{
	int referenced = 0;
	int we_locked = 0;

	*vm_flags = 0;
	if (page_mapped(page) &&  // アドレス空間にマップされているかどうか
       page_rmapping(page)) { // ?
       
		if (!is_locked && (!PageAnon(page) || PageKsm(page))) {
           // PG_locked をたてようとするt
			we_locked = trylock_page(page);
			if (!we_locked) {
				referenced++;
				goto out;
			}
		}
		if (unlikely(PageKsm(page)))
			referenced += page_referenced_ksm(page, mem_cont,
								vm_flags);
		else if (PageAnon(page))
			referenced += page_referenced_anon(page, mem_cont,
								vm_flags);
		else if (page->mapping)
			referenced += page_referenced_file(page, mem_cont,
								vm_flags);
		if (we_locked)
			unlock_page(page);

		if (page_test_and_clear_young(page))
			referenced++;
	}
out:

	return referenced;
}
```

page_referenced_* 属は下記の用に潜っていく

```
page_referenced_ksm    page_referenced_anon    page_referenced_file
               \                 |                   /
                 \               |                 /
                  +--------------+----------------+
                                 |
                                 |
                        page_referenced_one
                                 |
                                 |
                     ptep_clear_flush_young_notify
                                 |
                                 |
                        ptep_clear_flush_young
                                 |
                                 |
                      ptep_test_and_clear_young 
                                 |
                                 |   
     test_and_clear_bit(_PAGE_BIT_ACCESSED, (unsigned long *) &ptep->pte);
```

#### page_referenced_one

共通で呼び出される page_referenced_one の実装 

```c
/*
 * Subfunctions of page_referenced: page_referenced_one called
 * repeatedly from either page_referenced_anon or page_referenced_file.
 */
int page_referenced_one(struct page *page, struct vm_area_struct *vma,
			unsigned long address, unsigned int *mapcount,
			unsigned long *vm_flags)
{
	struct mm_struct *mm = vma->vm_mm;
	int referenced = 0;

    /* HugePage の場合 */
	if (unlikely(PageTransHuge(page))) {
		pmd_t *pmd;

		spin_lock(&mm->page_table_lock);
		/*
		 * rmap might return false positives; we must filter
		 * these out using page_check_address_pmd().
		 */
         /* pmd_t PMD があるかどうか */
		pmd = page_check_address_pmd(page, mm, address,
					     PAGE_CHECK_ADDRESS_PMD_FLAG);
		if (!pmd) {
			spin_unlock(&mm->page_table_lock);
			goto out;
		}

        /* mlock(2) されてるページは対象外? */
		if (vma->vm_flags & VM_LOCKED) {
			spin_unlock(&mm->page_table_lock);
			*mapcount = 0;	/* break early from loop */
			*vm_flags |= VM_LOCKED;
			goto out;
		}

		/* go ahead even if the pmd is pmd_trans_splitting() */
		if (pmdp_clear_flush_young_notify(vma, address, pmd))
			referenced++;
		spin_unlock(&mm->page_table_lock);
	} else {
        // 普通の呼び出しパスはこっちだろう

		pte_t *pte;      // typedef struct { pteval_t pte; } pte_t;
		spinlock_t *ptl;

		/*
		 * rmap might return false positives; we must filter
		 * these out using page_check_address().
		 */
         // mm->pgd から PGD -> PUD -> PMD と辿って PTE を取る
		pte = page_check_address(page, mm, address, &ptl, 0);
		if (!pte)
			goto out;

		if (vma->vm_flags & VM_LOCKED) {
			pte_unmap_unlock(pte, ptl);
			*mapcount = 0;	/* break early from loop */
			*vm_flags |= VM_LOCKED;
			goto out;
		}

        // PTE の Referenced ビットを落とす
        // https://www.cs.uaf.edu/2007/fall/cs301/lecture/11_30_cache.png
		if (ptep_clear_flush_young_notify(vma, address, pte)) {
			/*
			 * Don't treat a reference through a sequentially read
			 * mapping as such.  If the page has been used in
			 * another mapping, we will catch it; if this other
			 * mapping is already gone, the unmap path will have
			 * set PG_referenced or activated the page.
			 */
			if (likely(!VM_SequentialReadHint(vma)))
				referenced++;
		}
		pte_unmap_unlock(pte, ptl);
	}

	/* Pretend the page is referenced if the task has the
	   swap token and is in the middle of a page fault. */
	if (mm != current->mm && has_swap_token(mm) &&
			rwsem_is_locked(&mm->mmap_sem))
		referenced++;

	(*mapcount)--;

	if (referenced)
		*vm_flags |= vma->vm_flags;
out:
	return referenced;
}
```

pte_young は pte_flags(pte) & _PAGE_ACCESSED の意らしい

### activate_page

Inactive -> Active の LRU 移動

 * 対象の page から zone を選ぶ
 * page が file か anon かを選ぶ
 * SetPageActive する

```c
/*
 * FIXME: speed this up?
 */
void activate_page(struct page *page)
{
	struct zone *zone = page_zone(page);

    // LRU は zone ごとにあるので zone->lru_lock を取る
	spin_lock_irq(&zone->lru_lock);
	if (PageLRU(page)    && // なんだっけ?
       !PageActive(page) && // 既に Active なら何もしない
       !PageUnevictable(page)) {

       // file LRU に入れるべきかどうかを選択
       // return !PageSwapBacked(page);
		int file = page_is_file_cache(page);

        // LRU_INACTIVE_FILE か LRU_INACTIVE_ANON かを選択
		int lru = page_lru_base_type(page);

        // LRU からページを消す
		del_page_from_lru_list(zone, page, lru);

        // Active
		SetPageActive(page);
		lru += LRU_ACTIVE;
		add_page_to_lru_list(zone, page, lru);
		__count_vm_event(PGACTIVATE);

		update_page_reclaim_stat(zone, page, file, 1);
	}
	spin_unlock_irq(&zone->lru_lock);
}
```

### shrink_active_list

shrink_mem_cgroup_zone の過程で呼び出される

 * l_hold に deactivate 候補のページを isolate_pages で集める
 * l_active にやっぱり active に戻すページを集める
 * l_inactive に deactivate 対象のページを集める

```c
static void shrink_active_list(unsigned long nr_pages,
			       struct mem_cgroup_zone *mz,
			       struct scan_control *sc,
			       int priority, int file)
{
	unsigned long nr_taken;
	unsigned long pgscanned;
	unsigned long vm_flags;

    /* 関数無いでページを一時的に保持しておくようのリスト */
	LIST_HEAD(l_hold);	/* The pages which were snipped off */
	LIST_HEAD(l_active);
	LIST_HEAD(l_inactive);
	struct page *page;
	struct zone_reclaim_stat *reclaim_stat = get_reclaim_stat(mz);
	unsigned long nr_rotated = 0;
	int order = 0;
	struct zone *zone = mz->zone;

	if (!COMPACTION_BUILD)
		order = sc->order;

    // pagevecs ( per_cpu のリストの page 群) をLRU に移動させる?
	lru_add_drain();
	spin_lock_irq(&zone->lru_lock);

    // nr_pages 分スキャンして deactivate 候補のページを探す
    // isolate_pages はなかなか手強い
	nr_taken = isolate_pages(nr_pages, mz, &l_hold,
				 &pgscanned, order,
				 ISOLATE_ACTIVE, 1, file);

	if (global_reclaim(sc))
		zone->pages_scanned += pgscanned;

	reclaim_stat->recent_scanned[file] += nr_taken;

	__count_zone_vm_events(PGREFILL, zone, pgscanned);
	if (file)
		__mod_zone_page_state(zone, NR_ACTIVE_FILE, -nr_taken);
	else
		__mod_zone_page_state(zone, NR_ACTIVE_ANON, -nr_taken);
	__mod_zone_page_state(zone, NR_ISOLATED_ANON + file, nr_taken);
	spin_unlock_irq(&zone->lru_lock);

    // l_hold を走査する
	while (!list_empty(&l_hold)) {
		cond_resched();
		page = lru_to_page(&l_hold);
		list_del(&page->lru);

		if (unlikely(!page_evictable(page, NULL))) {
			putback_lru_page(page);
			continue;
		}

        /* Referenced ビットが立っているかどうかを見る */
		if (page_referenced(page, 0, sc->target_mem_cgroup, &vm_flags)) {
			nr_rotated += hpage_nr_pages(page);
			/*
			 * Identify referenced, file-backed active pages and
			 * give them one more trip around the active list. So
			 * that executable code get better chances to stay in
			 * memory under moderate memory pressure.  Anon pages
			 * are not likely to be evicted by use-once streaming
			 * IO, plus JVM can create lots of anon VM_EXEC pages,
			 * so we ignore them here.
			 */
             // ファイル backed な active ページを active list に戻す
             // 実行コードがオンメモリに残りやすい
             // anon ページは対象外
             / JVM は VM_EXEC なページを大量に生成する。無視
			if ((vm_flags & VM_EXEC) && page_is_file_cache(page)) {
				list_add(&page->lru, &l_active);
				continue;
			}
		}

        /* Active => Inactive へ移す候補に入れる */
		ClearPageActive(page);	/* we are de-activating */
		list_add(&page->lru, &l_inactive);
	}

	/*
	 * Move pages back to the lru list.
	 */
	spin_lock_irq(&zone->lru_lock);
	/*
	 * Count referenced pages from currently used mappings as rotated,
	 * even though only some of them are actually re-activated.  This
	 * helps balance scan pressure between file and anonymous pages in
	 * get_scan_ratio.
	 */
	reclaim_stat->recent_rotated[file] += nr_rotated;

	move_active_pages_to_lru(zone, &l_active,
						LRU_ACTIVE + file * LRU_FILE);
	move_active_pages_to_lru(zone, &l_inactive,
						LRU_BASE   + file * LRU_FILE);
	__mod_zone_page_state(zone, NR_ISOLATED_ANON + file, -nr_taken);
	spin_unlock_irq(&zone->lru_lock);
	trace_mm_pagereclaim_shrinkactive(pgscanned, file, priority);  
}
```

### isolate_lru_page

```c
/*
 * zone->lru_lock is heavily contended.  Some of the functions that
 * shrink the lists perform better by taking out a batch of pages
 * and working on them outside the LRU lock.
 *
 * For pagecache intensive workloads, this function is the hottest
 * spot in the kernel (apart from copy_*_user functions).
 *
 * Appropriate locks must be held before calling this function.
 *
 * @nr_to_scan:	The number of pages to look through on the list.
 * @src:	The LRU list to pull pages off.
 * @dst:	The temp list to put pages on to.
 * @scanned:	The number of pages that were scanned.
 * @order:	The caller's attempted allocation order
 * @mode:	One of the LRU isolation modes
 * @file:	True [1] if isolating file [!anon] pages
 *
 * returns how many pages were moved onto *@dst.
 */
static unsigned long isolate_lru_pages(unsigned long nr_to_scan,
		struct list_head *src, struct list_head *dst,
		unsigned long *scanned, int order, isolate_mode_t mode,
		int file)
{
	unsigned long nr_taken = 0;
	unsigned long nr_lumpy_taken = 0, nr_lumpy_dirty = 0, nr_lumpy_failed = 0;
	unsigned long scan;

	for (scan = 0; scan < nr_to_scan && !list_empty(src); scan++) {
		struct page *page;
		unsigned long pfn;
		unsigned long end_pfn;
		unsigned long page_pfn;
		int zone_id;

		page = lru_to_page(src);
		prefetchw_prev_lru_page(page, src, flags);

		VM_BUG_ON(!PageLRU(page));

		switch (__isolate_lru_page(page, mode, file)) {
		case 0:
			mem_cgroup_lru_del(page);
			list_move(&page->lru, dst);
			nr_taken += hpage_nr_pages(page);
			break;

		case -EBUSY:
			/* else it is being freed elsewhere */
			list_move(&page->lru, src);
			continue;

		default:
			BUG();
		}

		if (!order)
			continue;

		/*
		 * Attempt to take all pages in the order aligned region
		 * surrounding the tag page.  Only take those pages of
		 * the same active state as that tag page.  We may safely
		 * round the target page pfn down to the requested order
		 * as the mem_map is guarenteed valid out to MAX_ORDER,
		 * where that page is in a different zone we will detect
		 * it from its zone id and abort this block scan.
		 */
		zone_id = page_zone_id(page);
		page_pfn = page_to_pfn(page);
		pfn = page_pfn & ~((1 << order) - 1);
		end_pfn = pfn + (1 << order);
		for (; pfn < end_pfn; pfn++) {
			struct page *cursor_page;

			/* The target page is in the block, ignore it. */
			if (unlikely(pfn == page_pfn))
				continue;

			/* Avoid holes within the zone. */
			if (unlikely(!pfn_valid_within(pfn)))
				break;

			cursor_page = pfn_to_page(pfn);

			/* Check that we have not crossed a zone boundary. */
			if (unlikely(page_zone_id(cursor_page) != zone_id))
				break;

			/*
			 * If we don't have enough swap space, reclaiming of
			 * anon page which don't already have a swap slot is
			 * pointless.
			 */
			if (nr_swap_pages <= 0 && PageAnon(cursor_page) &&
			    !PageSwapCache(cursor_page))
				break;

			if (__isolate_lru_page(cursor_page, mode, file) == 0) {
				unsigned int isolated_pages;

				mem_cgroup_lru_del(cursor_page);
				list_move(&cursor_page->lru, dst);
				isolated_pages = hpage_nr_pages(page);
				nr_taken += isolated_pages;
				nr_lumpy_taken += isolated_pages;
				if (PageDirty(cursor_page))
					nr_lumpy_dirty += isolated_pages;
				scan++;
				pfn += isolated_pages - 1;
			} else {
				/*
				 * Check if the page is freed already.
				 *
				 * We can't use page_count() as that
				 * requires compound_head and we don't
				 * have a pin on the page here. If a
				 * page is tail, we may or may not
				 * have isolated the head, so assume
				 * it's not free, it'd be tricky to
				 * track the head status without a
				 * page pin.
				 */
				if (!PageTail(cursor_page) &&
				    !atomic_read(&cursor_page->_count))
					continue;
				break;
			}
		}

		/* If we break out of the loop above, lumpy reclaim failed */
		if (pfn < end_pfn)
			nr_lumpy_failed++;
	}

	*scanned = scan;

	trace_mm_vmscan_lru_isolate(order,
			nr_to_scan, scan,
			nr_taken,
			nr_lumpy_taken, nr_lumpy_dirty, nr_lumpy_failed,
			mode);
	return nr_taken;
}
```