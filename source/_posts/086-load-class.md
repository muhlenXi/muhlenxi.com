---
title: 『底层探索』12 - 初探类加载
toc: true
donate: false
tags: []
date: 2020-10-14 09:56:45
categories: [底层探索]
---

当可执行文件加载到内存后，dyld 会执行一系列的链接和初始化的流程，这其中也涉及到了类的加载，今天探索类是如何被加载的？

<!-- more -->

## dyld and runtime

在 dyld 的 `ImageLoader::recursiveInitialization` 方法中会递归初始化 App 依赖的库。初始化是通过调用 `ImageLoaderMachO::doInitialization` 方法来完成的。

在 `doInitialization` 方法中会调用以下的 2 个方法：

- `doImageInit` 初始化 Image（映像），是通过调用库中的 `init` 方法来完成的。
- `doModInitFunctions` 来执行库中的全局 C++ 对象构造函数、执行 `_attribute_(constructor)` 定义的函数。

那么对于 `libobjc.A.dylib` 动态库来说，也会执行它的 `init` 方法，我们找找它的实现。

```c
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    runtime_init();
    exception_init();
    cache_init();
    _imp_implementationWithBlock_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);  // 注册通知

#if __OBJC2__
    didCallDyldNotifyRegister = true;
#endif
}
```

在这个初始化方法中，通过全局变量保证该方法只会被调用一次。然后调用了 `环境变量`、`pthread`、`c++ 静态方法`、`runtime`、`异常`、`缓存` 等初始化方法。其中 `_imp_implementationWithBlock_init` 仅仅在 `TARGET_OS_OSX` 平台才有用。
这个方法中还有一个值得注意的地方就是调用了 `_dyld_objc_notify_register` 方法来注册通知。

在前篇文章中，我们知道 `_dyld_objc_notify_register` 是在 `dyld` 中实现的。

```c
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped)
{
	dyld::registerObjCNotifiers(mapped, init, unmapped);
}
```

registerObjCNotifiers 的实现如下：

```c
void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
{
	// record functions to call
	sNotifyObjCMapped	= mapped;
	sNotifyObjCInit		= init;
	sNotifyObjCUnmapped = unmapped;

	// call 'mapped' function with all images mapped so far
	try {
		notifyBatchPartial(dyld_image_state_bound, true, NULL, false, true);
	}
	catch (const char* msg) {
		// ignore request to abort during registration
	}

	// <rdar://problem/32209809> call 'init' function on all images already init'ed (below libSystem)
	for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {
		ImageLoader* image = *it;
		if ( (image->getState() == dyld_image_state_initialized) && image->notifyObjC() ) {
			dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
			(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());
		}
	}
}

```

通过上述三个方法，可以得到一下关系：

- 变量 sNotifyObjCMapped 的值是 map_images
- 变量 sNotifyObjCInit 的值是 load_images
- 变量 sNotifyObjCUnmapped 的值是 unmap_image

`doInitialization` 方法会间接调用 `registerObjCNotifiers` 方法，`registerObjCNotifiers` 方法中会调用 `notifyBatchPartial` 方法，在 `notifyBatchPartial` 实现中，我们发现了 sNotifyObjCMapped 会被调用，从而调用了 `map_images` 方法。

当 `doInitialization` 方法执行完毕后，会接着调用 `notifySingle` 方法，在 `notifySingle` 的实现中，sNotifyObjCInit 会被调用，从而调用了 `load_images` 方法。因此 `map_images` 会在 `load_images` 之前执行。

`sNotifyObjCUnmapped` 是在 `void removeImage(ImageLoader* image)` 方法中调用的, 也就是说 `unmap_image` 在这个方法中调用的。

用一张图可以说明上述的调用关系。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/doInit.png)

## map_images

下面我们看 `map_images` 方法中做了哪些事情。

```c
void
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

在这个方法中调用了 `map_images_nolock`，继续探索这个方法。在研究的源码中，我们可以使用 `command option left` 快捷键将无关代码收起来，这样可以使代码流程更清晰。

```c
void 
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[])
{
    static bool firstTime = YES;
    header_info *hList[mhCount];
    uint32_t hCount;
    size_t selrefCount = 0;

    if (firstTime) {
        preopt_init();
    }
    
    if (PrintImages) {...}
    
    hCount = 0;

    // Count classes. Size various table based on the total.
    int totalClasses = 0;
    int unoptimizedTotalClasses = 0;
    {
        uint32_t i = mhCount;
        while (i--) {
            const headerType *mhdr = (const headerType *)mhdrs[i];

            auto hi = addHeader(mhdr, mhPaths[i], totalClasses, unoptimizedTotalClasses);
            if (!hi) {
                // no objc data in this entry
                continue;
            }
            
            if (mhdr->filetype == MH_EXECUTE) {
                // Size some data structures based on main executable's size
#if __OBJC2__
                size_t count;
                _getObjc2SelectorRefs(hi, &count);
                selrefCount += count;
                _getObjc2MessageRefs(hi, &count);
                selrefCount += count;
#else
                _getObjcSelectorRefs(hi, &selrefCount);
#endif
                
#if SUPPORT_GC_COMPAT
                // Halt if this is a GC app.
                if (shouldRejectGCApp(hi)) {...}
#endif
            }
            
            hList[hCount++] = hi;
            
            if (PrintImages) {...}
        }
    }
if (firstTime) {
        sel_init(selrefCount);
        arr_init();

#if SUPPORT_GC_COMPAT
            for (uint32_t i = 0; i < hCount; i++) {
            auto hi = hList[i];
            auto mh = hi->mhdr();
            if (mh->filetype != MH_EXECUTE  &&  shouldRejectGCImage(mh)) {...}
        }
#endif

#if TARGET_OS_OSX
        if (dyld_get_program_sdk_version() < DYLD_MACOSX_VERSION_10_13) {...}

        for (uint32_t i = 0; i < hCount; i++) {...}
#endif

    }

    if (hCount > 0) {
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);
    }

    firstTime = NO;
    
    // Call image load funcs after everything is set up.
    for (auto func : loadImageFuncs) {
        for (uint32_t i = 0; i < mhCount; i++) {
            func(mhdrs[i]);
        }
    }
}
```

这段代码主要做了以下几件事情：

- 初始化 selector 表。
- 初始化 AutoreleasePoolPage、SideTablesMap 和 objc_associations 表。
- read_images
- 调用 class（实现了 load 方法） 的 load 方法。

接着我们看看 `_read_images` 方法。

```c
void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
    // 源码太长，未摘录。
}
```

## load_images

## 后记

此文未完成。

我是穆哥，卖码维生的一朵浪花。我们下回见。
