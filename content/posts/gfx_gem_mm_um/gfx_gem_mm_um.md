---
categories:
- work
date: 2014-12-23T14:02:00
tags:
- linux
- drm
- gfx
- i915
- gem
title: GEM Memory Address Management - User Mode
---

## 简介

这篇主要针对[上一篇]({{< ref gfx_gem_mm_km >}})
所讲的 `kernel` 提供给用户层提供的两个接口,
来分析下用户空间对 `buffer` 的管理,
这里以 `intel` 实现的 `buffer-manager` 为例子, 以下简称 `bufmgr` .

## Cache Buckets

这里 `bufmgr` 按照 `buffer` 的大小分为一个个的 `bucket` ,
相同大小的,或者大小相差不大的 `buffer` 会被放入到同一个 `bucket` 中,
我们来看大小是如何分布的:

```
unsigned long size, cache_max_size = 64 * 1024 * 1024;

add_bucket(bufmgr_gem, 4096);
add_bucket(bufmgr_gem, 4096 * 2);
add_bucket(bufmgr_gem, 4096 * 3);

/* Initialize the linked lists for BO reuse cache. */
for (size = 4 * 4096; size <= cache_max_size; size *= 2) {
	add_bucket(bufmgr_gem, size);

	add_bucket(bufmgr_gem, size + size * 1 / 4);
	add_bucket(bufmgr_gem, size + size * 2 / 4);
	add_bucket(bufmgr_gem, size + size * 3 / 4);
}
```

可以看出大小的分布是:
4k,8k,12k,16k,20k,24k,28k...
注意这里的分布不能按照等比的顺序,这样 `memory` 的占用会比较大.

## Allocation

当尝试分配一个 `buffer` ,我们首先尝试从 `cache` 中分配,
那么首先必须确定寻找的 `bucket` :

```
bucket = drm_intel_gem_bo_bucket_for_size(bufmgr_gem, size);
```

这里根据实际分配的大小,找到满足条件的最小的 `bucket`

```
for (i = 0; i < bufmgr_gem->num_buckets; i++) {
	struct drm_intel_gem_bo_bucket *bucket =
	    &bufmgr_gem->cache_bucket[i];
	if (bucket->size >= size) {
		return bucket;
	}
}
```

如果找到了满足条件的 `bucket` ,下面就是从中寻找合适的 `cache` `buffer` 了,
这里考虑到 `GPU` `cache` ,对于那些充当 `render-target` 的 `buffer`
从尾部开始寻找,因为释放时都是从尾部添加的.

```
DRMLISTFOREACHSAFEREVERSE(entry, temp, &bucket->head)
{
    bo_gem = DRMLISTENTRY(drm_intel_bo_gem,
		entry, head);

    if (bo_gem->bo.size >= size) {
		DRMLISTDEL(&bo_gem->head);
		alloc_from_cache = true;
		break;
    }
}
```

而对于非 `render-target` 的 `buffer` 选择从开头开始寻找:

```
DRMLISTFOREACHSAFE(entry, temp, &bucket->head)
{
    bo_gem = DRMLISTENTRY(drm_intel_bo_gem,
		entry, head);

    if ((bo_gem->bo.size >= size) &&
	!drm_intel_gem_bo_busy(&bo_gem->bo)) {
		DRMLISTDEL(&bo_gem->head);
		alloc_from_cache = true;
		break;
    }
}
```

注意这里除了保证大小满足条件,还需要保证该 `buffer` 当前处于 `unbusy` 的状态,
否则,我们可能分配了很多 `buffer` ,但是最终都在等待 `gpu` 处理完之后释放该
`buffer` .

由于是从 `cache` 中获得的 `buffer` ,
下面需要告诉 `kernel` 我们即将使用该 `buffer` ,
如果 `kernel` 已经将其占用的内存回收了,
我们需要真实释放掉该 `buffer`, 进行并重新寻找.

```
if (!drm_intel_gem_bo_madvise_internal
    (bufmgr_gem, bo_gem, I915_MADV_WILLNEED)) {
	drm_intel_gem_bo_free(&bo_gem->bo);
	drm_intel_gem_bo_cache_purge_bucket(bufmgr_gem,
					    bucket);
	goto retry;
}
```

其实这里还将该 `bucket` 中的其他已经被 `kernel` 释放的 `buffer` 释放掉.
这是在 `drm_intel_gem_bo_cache_purge_bucket` :

```
while (!DRMLISTEMPTY(&bucket->head)) {
	drm_intel_bo_gem *bo_gem;

	bo_gem = DRMLISTENTRY(drm_intel_bo_gem,
			      bucket->head.next, head);
	if (drm_intel_gem_bo_madvise_internal
	    (bufmgr_gem, bo_gem, I915_MADV_DONTNEED))
		break;

	DRMLISTDEL(&bo_gem->head);
	drm_intel_gem_bo_free(&bo_gem->bo);
}
```

如果最终我们没有从 `cache` `bucket` 中找到合适的 `buffer` ,
那么只能创建一个新的 `buffer` 了,
如果申请失败了,可能是由于的确没有足够的空间了,
这里,我们会将整个 `cache` `buckets` 都清空,
然后再重新尝试创建.

```
ret = drmIoctl(bufmgr_gem->fd,
	       DRM_IOCTL_I915_GEM_CREATE,
	       &create);

if (ret != 0) {
	/* If allocation failed, clear the cache and retry.
	 * Kernel has probably reclaimed any cached BOs already,
	 * but may as well retry after emptying the buckets.
	 */
	drm_intel_gem_empty_bo_cache(bufmgr_gem);

	VG_CLEAR(create);
	create.size = size;

	ret = drmIoctl(bufmgr_gem->fd,
		       DRM_IOCTL_I915_GEM_CREATE,
		       &create);

	if (ret != 0) {
		free(bo_gem);
		bo_gem = NULL;
		return NULL;
	}
}
```

## Free

当一个 `buffer` 的引用计数为0时,那么需要释放该 `buffer` 了,
这里做了两件事,首先是释放该 `buffer`, 之后是根据时间清理
`cache` `buckets`:

```
if (atomic_dec_and_test(&bo_gem->refcount)) {
	drm_intel_bufmgr_gem *bufmgr_gem =
	    (drm_intel_bufmgr_gem *) bo->bufmgr;
	struct timespec time;

	clock_gettime(CLOCK_MONOTONIC, &time);

	pthread_mutex_lock(&bufmgr_gem->lock);
	drm_intel_gem_bo_unreference_final(bo, time.tv_sec);
	drm_intel_gem_cleanup_bo_cache(bufmgr_gem, time.tv_sec);
	pthread_mutex_unlock(&bufmgr_gem->lock);
}
```

注意,这里的时间单位为秒.

释放一个 `buffer` 时,首先会通知 `kernel` 该 `buffer` 对应的内存可以被回收,
如果 `kernel` 没有立即回收该内存,那么会将该 `buffer` 放入到对应的 `bucket`
的尾部

```
if (bufmgr_gem->bo_reuse && bo_gem->reusable && bucket != NULL &&
    drm_intel_gem_bo_madvise_internal(bufmgr_gem, bo_gem,
				      I915_MADV_DONTNEED)) {
	bo_gem->free_time = time;

	bo_gem->name = NULL;
	bo_gem->validate_index = -1;

	DRMLISTADDTAIL(&bo_gem->head, &bucket->head);
```

否则,直接将该 `buffer` 释放.

```
} else {
	drm_intel_gem_bo_free(bo);
}
```

这里的目的主要是,在每次释放一个 `buffer` 之后
会遍历整个 `cache` `bucket` 将在之前1秒之前的所有的 `cache` 都释放掉:

```
for (i = 0; i < bufmgr_gem->num_buckets; i++) {
	struct drm_intel_gem_bo_bucket *bucket =
	    &bufmgr_gem->cache_bucket[i];

	while (!DRMLISTEMPTY(&bucket->head)) {
		drm_intel_bo_gem *bo_gem;

		bo_gem = DRMLISTENTRY(drm_intel_bo_gem,
				      bucket->head.next, head);
		if (time - bo_gem->free_time <= 1)
			break;

		DRMLISTDEL(&bo_gem->head);

		drm_intel_gem_bo_free(&bo_gem->bo);
	}
}
```

FIN.
