---
title: 『底层探索』11 - iOS App 启动流程分析一
toc: true
donate: false
tags: []
date: 2020-09-29 19:57:42
categories: [底层探索]
---

我们都知道 iOS App 程序的入口是 main 函数，其实在 main 函数之前还有很多步骤要做，做完才会调用我们的 main 函数进入主程序，今天就探索 main 函数之前的流程。

<!-- more -->

## 源代码 -> 可执行文件

探索前先了解下源代码到可执行文件主要经历了哪些流程。

对于平常应用开发，很少关注编译和链接过程。通常 IDE 一般都将编译和链接的过程一步完成，这种将编译和链接合并到一起的过程称为`构建`(Build)。

程序从源代码到可执行文件一般经历四个步骤，分别是 `预处理`（Preprocessing），`编译`（Compilation），`汇编`（Assembly），`链接`（Linking）。

如图简单描述了源代码到最终目标代码的过程。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/compiler.png)

## App 启动探索

要探索 App 启动是离不开 `DYLD` 的，DYLD(the dynamic link editor) 是 Apple 的动态链接器，是 Apple 操作系统中的重要组成部分，当系统内核做好程序准备工作后，就交由 dyld 完成剩余的工作。也就是内核读取完 `Mach-O` 的 header 后，就交给 DYLD 了。

不熟悉 Mach-O 文件的可以看看我前篇 [Mach-O 文件分析](https://www.muhlenxi.com/2020/09/28/084-mach-o/)。

新建一个包含 main 函数的 Single View Application 工程，在 main 函数入口那里打个断点，然后看看堆栈调用情况。

```shell
(lldb) bt
    frame #1: 0x00000001851f98f0 libdyld.dylib`start + 4
```

在 main 函数这里没发现有用信息，之前有了解到 `+load` 方法的调用时机会靠前些。我们在 `ViewController` 的 `+load` 那里打个断点，看看调用堆栈信息。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/loadplus.jpg)

通过这个调用堆栈，从下往上，可以得到一些有用信息。接下来我们开始我们的探索。

本文探索的环境是：Xcode12、iPhone X、ARM 64。

## dyld 探索

### _dyld_start

首先调用 `_dyld_start` 方法，google `dyld apple` 关键字可找到 [dyld](https://opensource.apple.com/tarballs/dyld/) 的开源代码。我们下载最新的 `dyld-750.6.tar.gz` 版本探索下。

由于 dyld 工程不能编译运行，我们只能根据关键词、关键代码及其注释来进行逻辑流程分析了。

【1】在 `dyld` 工程中，全局搜索 `_dyld_start` 关键字，找到了 arm64 对应的代码实现。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/dyld-start.jpg)

### dyldbootstrap::start

【2】图中的代码主要做了一些配置工作，然后跳转到 `dyldbootstrap::start` 方法，bootstrap 有引导的意思，可以猜测这个是 dyld 引导程序的入口。我们再搜索一下这个方法。

```c
uintptr_t start(const dyld3::MachOLoaded* appsMachHeader, int argc, const char* argv[],
				const dyld3::MachOLoaded* dyldsMachHeader, uintptr_t* startGlue)
{

    // Emit kdebug tracepoint to indicate dyld bootstrap has started <rdar://46878536>
    dyld3::kdebug_trace_dyld_marker(DBG_DYLD_TIMING_BOOTSTRAP_START, 0, 0, 0, 0);

	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
    rebaseDyld(dyldsMachHeader);

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
	uintptr_t appsSlide = appsMachHeader->getSlide();
	return dyld::_main((macho_header*)appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

### dyld::_main

【3】这里做了一些 dyld 的参数配置，最后调用了 `dyld::_main` 方法，按图索骥。

在 `dyld2.cpp` 中找到了定位到了这个方法。

```c
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
    // 函数体分布在 6191 - 6828 行，太长了，可以去看源码。
}
```

根据注释和代码命名，大概可以了解到主要做了以下的事情：

- 0、set context。设置上下文
- 1、configure process restrictions。配置进程
- 2、check environment variables。查看环境变量
- 3、get host info。获取当前程序的架构信息。
- 4、check shared region disable。检查共享缓存是否被禁用。
- 5、map shared cache。加载共享缓存。
- 6、instantiate ImageLoader for main executable。也就是生成主执行程序实例。
    - instantiateFromLoadedImage 方法调用 `instantiateMainExecutable` 方法生成主执行程序实例
    - sniffLoadCommands 加载 load commands。
- 7、load any inserted libraries。加载插入的动态库。
- 8、link main executable。链接主程序。
- 9、link any inserted libraries。链接插入的动态库。
- 10、Bind and notify for the inserted images。绑定插入的动态库并通知相应的观察者。
- 11、run all initializers。开始所有的初始化过程。 
- 12、find entry point for main executable。获取 main 函数入口地址并返回。

## 局部流程分析

### checkSharedRegionDisable

4、共享缓存是否禁用

```c
static void checkSharedRegionDisable(const dyld3::MachOLoaded* mainExecutableMH, uintptr_t mainExecutableSlide)
{
#if __MAC_OS_X_VERSION_MIN_REQUIRED
	// if main executable has segments that overlap the shared region,
	// then disable using the shared region
	if ( mainExecutableMH->intersectsRange(SHARED_REGION_BASE, SHARED_REGION_SIZE) ) {
		gLinkContext.sharedRegionMode = ImageLoader::kDontUseSharedRegion;
		if ( gLinkContext.verboseMapping )
			dyld::warn("disabling shared region because main executable overlaps\n");
	}
#if __i386__
	if ( !gLinkContext.allowEnvVarsPath ) {
		// <rdar://problem/15280847> use private or no shared region for suid processes
		gLinkContext.sharedRegionMode = ImageLoader::kUsePrivateSharedRegion;
	}
#endif
#endif
	// iOS cannot run without shared region
}
```

也就是说在 iOS 中不能禁用共享缓存，不然无法运行的。

### mapSharedCache

5、加载共享缓存

```c
static void mapSharedCache()
{
	dyld3::SharedCacheOptions opts;
	opts.cacheDirOverride	= sSharedCacheOverrideDir;
	opts.forcePrivate		= (gLinkContext.sharedRegionMode == ImageLoader::kUsePrivateSharedRegion);


#if __x86_64__ && !TARGET_OS_SIMULATOR
	opts.useHaswell			= sHaswell;
#else
	opts.useHaswell			= false;
#endif
	opts.verbose			= gLinkContext.verboseMapping;
	loadDyldCache(opts, &sSharedCacheLoadInfo);

	// update global state
	if ( sSharedCacheLoadInfo.loadAddress != nullptr ) {
		gLinkContext.dyldCache 								= sSharedCacheLoadInfo.loadAddress;
		dyld::gProcessInfo->processDetachedFromSharedRegion = opts.forcePrivate;
		dyld::gProcessInfo->sharedCacheSlide                = sSharedCacheLoadInfo.slide;
		dyld::gProcessInfo->sharedCacheBaseAddress          = (unsigned long)sSharedCacheLoadInfo.loadAddress;
		sSharedCacheLoadInfo.loadAddress->getUUID(dyld::gProcessInfo->sharedCacheUUID);
		dyld3::kdebug_trace_dyld_image(DBG_DYLD_UUID_SHARED_CACHE_A, sSharedCacheLoadInfo.path, (const uuid_t *)&dyld::gProcessInfo->sharedCacheUUID[0], {0,0}, {{ 0, 0 }}, (const mach_header *)sSharedCacheLoadInfo.loadAddress);
	}
}

bool loadDyldCache(const SharedCacheOptions& options, SharedCacheLoadInfo* results)
{
    results->loadAddress        = 0;
    results->slide              = 0;
    results->errorMessage       = nullptr;

#if TARGET_OS_SIMULATOR
    // simulator only supports mmap()ing cache privately into process
    return mapCachePrivate(options, results);
#else
    if ( options.forcePrivate ) {
        // mmap cache into this process only
        // 仅仅把共享缓存加载到当前进程中
        return mapCachePrivate(options, results);
    }
    else {
        // fast path: when cache is already mapped into shared region
        // 共享缓存只加载一次，如果当前进程存在共享缓存则跳过
        bool hasError = false;
        if ( reuseExistingCache(options, results) ) {
            hasError = (results->errorMessage != nullptr);
        } else {
            // slow path: this is first process to load cache
            hasError = mapCacheSystemWide(options, results);
        }
        return hasError;
    }
#endif
}
```

### instantiateFromLoadedImage

6、通过 `instantiateFromLoadedImage` 方法生成 `sMainExecutable` 主执行程序实例`instantiateFromLoadedImage` 方法的实现是这样的。

```c
static ImageLoaderMachO* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
	// try mach-o loader
	if ( isCompatibleMachO((const uint8_t*)mh, path) ) {
		ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext);
		// 添加到全局镜像表 sAllImages 中
		addImage(image);
		return (ImageLoaderMachO*)image;
	}
	
	throw "main executable not a known format";
}
```

`isCompatibleMachO` 是通过 Mach-O Header 中的信息来判断兼容性的。

这一步主要是创建 `ImageLoader`, ImageLoader 是一个抽象的用于加载指定的可执行格式文件的类。

`ImageLoaderMachO` 是 `ImageLoader` 的子类，是用来加载 `mach-o` 格式的文件。

调用 `ImageLoaderMachO::instantiateMainExecutable` 方法可以得到 imageLoader。这个方法主要是 `create image for main executable`。

### instantiateMainExecutable

```c
void initializeMainExecutable()
{
	// record that we've reached this step
	gLinkContext.startedInitializingMainExecutable = true;

	// run initialzers for any inserted dylibs
	ImageLoader::InitializerTimingList initializerTimes[allImagesCount()];
	initializerTimes[0].count = 0;
	const size_t rootCount = sImageRoots.size();
	if ( rootCount > 1 ) {
		for(size_t i=1; i < rootCount; ++i) {
			sImageRoots[i]->runInitializers(gLinkContext, initializerTimes[0]);
		}
	}
	
	// run initializers for main executable and everything it brings up 
	sMainExecutable->runInitializers(gLinkContext, initializerTimes[0]);
	
	// register cxa_atexit() handler to run static terminators in all loaded images when this process exits
	if ( gLibSystemHelpers != NULL ) 
		(*gLibSystemHelpers->cxa_atexit)(&runAllStaticTerminators, NULL, NULL);

	// dump info if requested
	if ( sEnv.DYLD_PRINT_STATISTICS )
		ImageLoader::printStatistics((unsigned int)allImagesCount(), initializerTimes[0]);
	if ( sEnv.DYLD_PRINT_STATISTICS_DETAILS )
		ImageLoaderMachO::printStatisticsDetails((unsigned int)allImagesCount(), initializerTimes[0]);
}
```

8、因为主执行程序是以 `ASLR` 的方式加载进内存的，所有需要 link 和 rebase，以修复内部地址的偏移。

9、所有插入的动态库的链接是调用 `link` 方法来完成的。

10、所有插入的动态库的 bind 是调用 image 的 `recursiveBind` 方法来完成的。

11、主执行程序的初始化是通过 `runInitializers` 来完成的，究竟要对哪些东西初始化呢，我们来探索下。

这里主要是对链接完毕的动态库和主执行程序，调用 `runInitializers` 方法，由于它们都是 `ImageLoader`, 下一步就是去 `ImageLoader` 声明的地方，看看 `runInitializers` 的实现。

在 `ImageLoader.cpp` 文件 602 行可以找到该方法的实现。

### runInitializers

```c
void ImageLoader::runInitializers(const LinkContext& context, InitializerTimingList& timingInfo)
{
	uint64_t t1 = mach_absolute_time();
	mach_port_t thisThread = mach_thread_self();
	ImageLoader::UninitedUpwards up;
	up.count = 1;
	up.imagesAndPaths[0] = { this, this->getPath() };
	processInitializers(context, thisThread, timingInfo, up);
	context.notifyBatch(dyld_image_state_initialized, false);
	mach_port_deallocate(mach_task_self(), thisThread);
	uint64_t t2 = mach_absolute_time();
	fgTotalInitTime += (t2 - t1);
}
```

这个方法又是调用 `processInitializers` 方法来初始化的。

### processInitializers

```c
void ImageLoader::processInitializers(const LinkContext& context, mach_port_t thisThread,
									 InitializerTimingList& timingInfo, ImageLoader::UninitedUpwards& images)
{
	uint32_t maxImageCount = context.imageCount()+2;
	ImageLoader::UninitedUpwards upsBuffer[maxImageCount];
	ImageLoader::UninitedUpwards& ups = upsBuffer[0];
	ups.count = 0;
	// Calling recursive init on all images in images list, building a new list of
	// uninitialized upward dependencies.
	for (uintptr_t i=0; i < images.count; ++i) {
		images.imagesAndPaths[i].first->recursiveInitialization(context, thisThread, images.imagesAndPaths[i].second, timingInfo, ups);
	}
	// If any upward dependencies remain, init them.
	if ( ups.count > 0 )
		processInitializers(context, thisThread, timingInfo, ups);
}
```

在这个方法里，对 image 调用了 `recursiveInitialization` 方法进行初始化，如果 image 还有向上还有依赖的 image，则对依赖的 image 进行调用 `processInitializers` 初始化。

### recursiveInitialization

```c
void ImageLoader::recursiveInitialization(const LinkContext& context, mach_port_t this_thread, const char* pathToInitialize,
										  InitializerTimingList& timingInfo, UninitedUpwards& uninitUps)
{
	recursive_lock lock_info(this_thread);
	recursiveSpinLock(lock_info);

	if ( fState < dyld_image_state_dependents_initialized-1 ) {
		uint8_t oldState = fState;
		// break cycles
		fState = dyld_image_state_dependents_initialized-1;
		try {
			// initialize lower level libraries first
			for(unsigned int i=0; i < libraryCount(); ++i) {
				ImageLoader* dependentImage = libImage(i);
				if ( dependentImage != NULL ) {
					// don't try to initialize stuff "above" me yet
					if ( libIsUpward(i) ) {
						uninitUps.imagesAndPaths[uninitUps.count] = { dependentImage, libPath(i) };
						uninitUps.count++;
					}
					else if ( dependentImage->fDepth >= fDepth ) {
						dependentImage->recursiveInitialization(context, this_thread, libPath(i), timingInfo, uninitUps);
					}
                }
			}
			
			// record termination order
			if ( this->needsTermination() )
				context.terminationRecorder(this);

			// let objc know we are about to initialize this image
			uint64_t t1 = mach_absolute_time();
			fState = dyld_image_state_dependents_initialized;
			oldState = fState;
			context.notifySingle(dyld_image_state_dependents_initialized, this, &timingInfo);
			
			// initialize this image
			bool hasInitializers = this->doInitialization(context);

			// let anyone know we finished initializing this image
			fState = dyld_image_state_initialized;
			oldState = fState;
			context.notifySingle(dyld_image_state_initialized, this, NULL);
			
			if ( hasInitializers ) {
				uint64_t t2 = mach_absolute_time();
				timingInfo.addTime(this->getShortName(), t2-t1);
			}
		}
		catch (const char* msg) {
			// this image is not initialized
			fState = oldState;
			recursiveSpinUnLock();
			throw;
		}
	}
	
	recursiveSpinUnLock();
}
```

在这里是调用 `doInitialization(context)` 方法来初始化 image 的。探索下这个方法。

### doInitialization

```c
bool ImageLoaderMachO::doInitialization(const LinkContext& context)
{
	CRSetCrashLogMessage2(this->getPath());

	// mach-o has -init and static initializers
	// 调用 init 方法进行初始化
	doImageInit(context);
	// 调用 C++ 构造方法
	doModInitFunctions(context);
	
	CRSetCrashLogMessage2(NULL);
	
	return (fHasDashInit || fHasInitializers);
}
```

在 `recursiveInitialization` 这个方法中同时找到了关键线索 `notifySingle`，全局搜一下这个方法的实现。

在 `dyld2.cpp` 的 909 行找到了实现。

### notifySingle

```c
static void notifySingle(dyld_image_states state, const ImageLoader* image, ImageLoader::InitializerTimingList* timingInfo)
{
	//dyld::log("notifySingle(state=%d, image=%s)\n", state, image->getPath());
	std::vector<dyld_image_state_change_handler>* handlers = stateToHandlers(state, sSingleHandlers);
	if ( handlers != NULL ) {
		dyld_image_info info;
		info.imageLoadAddress	= image->machHeader();
		info.imageFilePath		= image->getRealPath();
		info.imageFileModDate	= image->lastModified();
		for (std::vector<dyld_image_state_change_handler>::iterator it = handlers->begin(); it != handlers->end(); ++it) {
			const char* result = (*it)(state, 1, &info);
			if ( (result != NULL) && (state == dyld_image_state_mapped) ) {
				//fprintf(stderr, "  image rejected by handler=%p\n", *it);
				// make copy of thrown string so that later catch clauses can free it
				const char* str = strdup(result);
				throw str;
			}
		}
	}
	if ( state == dyld_image_state_mapped ) {
		// <rdar://problem/7008875> Save load addr + UUID for images from outside the shared cache
		if ( !image->inSharedCache() ) {
			dyld_uuid_info info;
			if ( image->getUUID(info.imageUUID) ) {
				info.imageLoadAddress = image->machHeader();
				addNonSharedCacheImageUUID(info);
			}
		}
	}
	if ( (state == dyld_image_state_dependents_initialized) && (sNotifyObjCInit != NULL) && image->notifyObjC() ) {
		uint64_t t0 = mach_absolute_time();
		dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
		(*sNotifyObjCInit)(image->getRealPath(), image->machHeader());
		uint64_t t1 = mach_absolute_time();
		uint64_t t2 = mach_absolute_time();
		uint64_t timeInObjC = t1-t0;
		uint64_t emptyTime = (t2-t1)*100;
		if ( (timeInObjC > emptyTime) && (timingInfo != NULL) ) {
			timingInfo->addTime(image->getShortName(), timeInObjC);
		}
	}
    // mach message csdlc about dynamically unloaded images
	if ( image->addFuncNotified() && (state == dyld_image_state_terminated) ) {
		notifyKernel(*image, false);
		const struct mach_header* loadAddress[] = { image->machHeader() };
		const char* loadPath[] = { image->getPath() };
		notifyMonitoringDyld(true, 1, loadAddress, loadPath);
	}
}
```

着重看一下第 3 个 if 判断，当 state 为 `dyld_image_state_dependents_initialized` ，image `notifyObjC()` 获取的方法为 YES，并且 `sNotifyObjCInit` 不为空，然后调用了这个回调。

通过前面的堆栈信息得知，下一步是 `load_images`, 那么这个是怎么触发的呢？ 没思路的时候，先看看这个回调是怎么被赋值的？经过全文检索，发现有一个在 `registerObjCNotifiers` 方法中对 `sNotifyObjCInit` 赋值了，是这个方法的第二个参数。

### registerObjCNotifiers

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

这个方法的第二个参数赋值了 `sNotifyObjCInit`, 那么这个 `registerObjCNotifiers` 的调用时机在哪里了？继续全文检索。

在 `dyldAPIs.cpp` 中找到了 `_dyld_objc_notify_register` 方法会调用 `registerObjCNotifiers`。那 `_dyld_objc_notify_register` 又是被谁调用的呢？全文检索没找到有用信息。事情似乎陷入了僵局。

这个函数所在的文件名提供了部分思路，是 API，那么就有可能被其他模块的代码调用了。去下个 `_dyld_objc_notify_register` 符号断点。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/dyld_notify.jpg)

项目运行到断点处停下了，我们看看此时的调用堆栈。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/dyld-notify-stack.jpg)

根据堆栈中的信息，也就是说在 `notifySingle` 之前的共享库初始化阶段就设置了通知回调。`libSystem` 和 `libdispatch` 代码都有开源。

### libSystem_initializer

下载 [Libsystem](https://opensource.apple.com/tarballs/Libsystem/) 工程后搜一下 `libSystem_initializer` 的实现。

在 `init.c` 文件中找到了`libSystem_initializer` 的实现，代码行数也不少，我们摘录下关键信息。

```c
libSystem_initializer(int argc,
		      const char* argv[],
		      const char* envp[],
		      const char* apple[],
		      const struct ProgramVars* vars)
{
    __libkernel_init(&libkernel_funcs, envp, apple, vars);
    __libplatform_init(NULL, envp, apple, vars);
    __pthread_init(&libpthread_funcs, envp, apple, vars);
    _libc_initializer(&libc_funcs, envp, apple, vars);
    __malloc_init(apple);
    _dyld_initializer();
    libdispatch_init();
}
```

在这个方法中调用了许多基础模块的 init 方法，上面列出的都是我们熟悉的模块。下一步看看 `libdispatch_init` 方法又初始化了哪些东西。

### libdispatch_init

下载 [libdispatch](https://opensource.apple.com/tarballs/libdispatch/) 工程后搜一下 `libdispatch_init` 的实现。

在 `queue.c` 文件中找到了`libdispatch_init` 的实现，代码行数也不少，我们摘录下关键信息。

```c
DISPATCH_EXPORT DISPATCH_NOTHROW
void
libdispatch_init(void)
{
	_dispatch_hw_config_init();
	_dispatch_time_init();
	_dispatch_vtable_init();
	_os_object_init();
	_voucher_init();
	_dispatch_introspection_init();
}
```

看看 `_os_object_init` 的实现是咋样的？原来在这里调用了 `_objc_init` 方法。

```c
_os_object_init(void)
{
	_objc_init();
	// 后面的代码省略了
}
```

再看看 `_objc_init` 的方法实现，直接去 `objc4` 工程中去找。在 `objc-os.mm` 文件中找到了实现。

### _objc_init

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

在 `_objc_init` 方法里又调用了 `_dyld_objc_notify_register` 方法，传入了 `load_images` 方法的地址。`load_images` 方法勾起了我的好奇心。

### load_images

```c
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    if (!didInitialAttachCategories && didCallDyldNotifyRegister) {
        didInitialAttachCategories = true;
        loadAllCategories();
    }

    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

不简单，在 `load_images` 方法中，调用了 `loadAllCategories`、`prepare_load_methods` 和 `call_load_methods` 这 3 个关键方法。根据注释，这里就到了我们的 `+load`方法的断点的地方。这个方法后面的流程我们改天再探索。

## 总结

至此我们的调用线索梳理清楚了。我们来总结下完整的流程：

- 1、当 App 可执行文件加载到内存后，Dyld 开始工作了，从调用 `_dyld_start` 方法开始。
- 2、接着 dyld 执行 `dyldbootstrap::start` 方法，这个方法参数设置完会调用 `dyld::_main` 方法。
- 3、在 _main 中，dyld 调用了 `instantiateFromLoadedImage` 创建了主可执行程序 实例。
- 4、主可执行程序创建后，就调用 `runInitializers` 开始初始化流程。
- 5、然后对可执行程序依赖的库进行 `recursiveInitialization` 操作。
    - 当遇到了 `libsystem` 时，会调用 `libSystem_initializer` 方法进行初始化。
    - `libSystem_initializer` 则会调用 `_dyld_initializer` 和 `libdispatch_init` 初始化方法。
    - 在 `libdispatch_init` 方法中又间接通过 `_os_object_init` 方法调用了 `_objc_init` 初始化方法。
    - `_objc_init` 方法中调用了 `_dyld_objc_notify_register` 方法设置了 sNotifyObjCInit 的回调地址。
    - 当 `libobjc` 初始化完毕后将会调用 `sNotifyObjCInit` 回调，也就是调用了 `load_images` 方法。
- 6、当 `runInitializers` 初始化流程完毕后，会调用 `ImageLoaderMachO::getEntryFromLC_MAIN()` 得到 `main` 函数的地址，然后调用 `main` 函数。


## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。 
