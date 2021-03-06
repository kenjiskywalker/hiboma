# [posix_fadvise(2)](http://kazmax.zpp.jp/cmd/p/posix_fadvise.2.html)

 * POSIX_FADV_NORMAL, POSIX_FADV_RANDOM, POSIX_FADV_SEQUENTIAL
   * backing_dev_info の ra_pages (readahed) の数値をいじってる
   * force_page_cache_readahead ???
 * POSIX_FADV_DONTNEED
   * bdi_write_congested ???
   * [filemap_flush](http://lxr.free-electrons.com/source/mm/filemap.c?v=2.6.32#L256])
   * [invalidate_mapping_pages](http://lxr.free-electrons.com/source/mm/truncate.c?v=2.6.32#306)
     * `unlocked pages` を破棄する。 `truncate_inode_pages` だと locked な ページも破棄できると※
     * trylock_page でロックの有無を見てるのかな?
   * read(2) -> ページキャッシュ埋まる -> POSIX_FADV_DONTNEED でページキャッシュを破棄
 * POSIX_FADV_NOREUSE はどう動くのかが分からん

## 2.6.9

```
/*
 * POSIX_FADV_WILLNEED could set PG_Referenced, and POSIX_FADV_NOREUSE could
 * deactivate the pages and clear PG_Referenced.
 */
asmlinkage long sys_fadvise64_64(int fd, loff_t offset, loff_t len, int advice)
{
	struct file *file = fget(fd);
	struct address_space *mapping;
	struct backing_dev_info *bdi;
	loff_t endbyte;
	pgoff_t start_index;
	pgoff_t end_index;
	unsigned long nrpages;
	int ret = 0;

	if (!file)
		return -EBADF;

	mapping = file->f_mapping;
	if (!mapping || len < 0) {
		ret = -EINVAL;
		goto out;
	}

	/* Careful about overflows. Len == 0 means "as much as possible" */
	endbyte = offset + len;
	if (!len || endbyte < len)
		endbyte = -1;

	bdi = mapping->backing_dev_info;

	switch (advice) {
	case POSIX_FADV_NORMAL:
		file->f_ra.ra_pages = bdi->ra_pages;
		break;
	case POSIX_FADV_RANDOM:
		file->f_ra.ra_pages = 0; // 先読みしない
		break;
	case POSIX_FADV_SEQUENTIAL:
		file->f_ra.ra_pages = bdi->ra_pages * 2; // 2倍 !!!
		break;
	case POSIX_FADV_WILLNEED:
	case POSIX_FADV_NOREUSE:
		if (!mapping->a_ops->readpage) {
			ret = -EINVAL;
			break;
		}

		/* First and last PARTIAL page! */
		start_index = offset >> PAGE_CACHE_SHIFT;
		end_index = (endbyte-1) >> PAGE_CACHE_SHIFT;

		/* Careful about overflow on the "+1" */
		nrpages = end_index - start_index + 1;
		if (!nrpages)
			nrpages = ~0UL;

        // なんか強制的に先読みできるぽい
		ret = force_page_cache_readahead(mapping, file,
				start_index,
				max_sane_readahead(nrpages));
		if (ret > 0)
			ret = 0;
		break;
	case POSIX_FADV_DONTNEED:
        
		if (!bdi_write_congested(mapping->backing_dev_info))
			filemap_flush(mapping);

		/* First and last FULL page! */
		start_index = (offset + (PAGE_CACHE_SIZE-1)) >> PAGE_CACHE_SHIFT;
		end_index = (endbyte >> PAGE_CACHE_SHIFT);

		if (end_index > start_index)
			invalidate_mapping_pages(mapping, start_index, end_index-1);
		break;
	default:
		ret = -EINVAL;
	}
out:
	fput(file);
	return ret;
}
```

## 2.6.32

 ```c
/*
 * POSIX_FADV_WILLNEED could set PG_Referenced, and POSIX_FADV_NOREUSE could
 * deactivate the pages and clear PG_Referenced.
 */
SYSCALL_DEFINE(fadvise64_64)(int fd, loff_t offset, loff_t len, int advice)
{
	struct file *file = fget(fd);
	struct address_space *mapping;
	struct backing_dev_info *bdi;
	loff_t endbyte;			/* inclusive */
	pgoff_t start_index;
	pgoff_t end_index;
	unsigned long nrpages;
	int ret = 0;

	if (!file)
		return -EBADF;

    // FIFO には使えない
	if (S_ISFIFO(file->f_path.dentry->d_inode->i_mode)) {
		ret = -ESPIPE;
		goto out;
	}

	mapping = file->f_mapping;
	if (!mapping || len < 0) {
		ret = -EINVAL;
		goto out;
	}

	if (mapping->a_ops->get_xip_mem) {
		switch (advice) {
		case POSIX_FADV_NORMAL:
		case POSIX_FADV_RANDOM:
		case POSIX_FADV_SEQUENTIAL:
		case POSIX_FADV_WILLNEED:
		case POSIX_FADV_NOREUSE:
		case POSIX_FADV_DONTNEED:
			/* no bad return value, but ignore advice */
			break;
		default:
			ret = -EINVAL;
		}
		goto out;
	}

	/* Careful about overflows. Len == 0 means "as much as possible" */
	endbyte = offset + len;
	if (!len || endbyte < len)
		endbyte = -1;
	else
		endbyte--;		/* inclusive */

	bdi = mapping->backing_dev_info;

	switch (advice) {
	case POSIX_FADV_NORMAL:
		file->f_ra.ra_pages = bdi->ra_pages;
		spin_lock(&file->f_lock);
		file->f_mode &= ~FMODE_RANDOM;
		spin_unlock(&file->f_lock);
		break;
	case POSIX_FADV_RANDOM:
		spin_lock(&file->f_lock);
		file->f_mode |= FMODE_RANDOM;            // 0 にはしないのかな?
		spin_unlock(&file->f_lock);
		break;
	case POSIX_FADV_SEQUENTIAL: 
		file->f_ra.ra_pages = bdi->ra_pages * 2; // 2倍
		spin_lock(&file->f_lock);
		file->f_mode &= ~FMODE_RANDOM;
		spin_unlock(&file->f_lock);
		break;
	case POSIX_FADV_WILLNEED:
		if (!mapping->a_ops->readpage) {
			ret = -EINVAL;
			break;
		}

		/* First and last PARTIAL page! */
		start_index = offset >> PAGE_CACHE_SHIFT;
		end_index = endbyte >> PAGE_CACHE_SHIFT;

		/* Careful about overflow on the "+1" */
		nrpages = end_index - start_index + 1;
		if (!nrpages)
			nrpages = ~0UL;
		
		ret = force_page_cache_readahead(mapping, file,
				start_index,
				nrpages);
		if (ret > 0)
			ret = 0;
		break;
	case POSIX_FADV_NOREUSE:
         // なんもしてないぞ
		break;
	case POSIX_FADV_DONTNEED:
		if (!bdi_write_congested(mapping->backing_dev_info))
			filemap_flush(mapping);

		/* First and last FULL page! */
		start_index = (offset+(PAGE_CACHE_SIZE-1)) >> PAGE_CACHE_SHIFT;
		end_index = (endbyte >> PAGE_CACHE_SHIFT);

		if (end_index >= start_index)
			invalidate_mapping_pages(mapping, start_index,
						end_index);
		break;
	default:
		ret = -EINVAL;
	}
out:
	fput(file);
	return ret;
}
``` 
