# ART GC介绍
## 1. GC Type

art/runtime/gc/collector/gc\_type.h

>  // Placeholder for when no GC has been performed.
>
>  kGcTypeNone,
>
>  // Sticky mark bits GC that attempts to only free objects allocated since the last GC.
>
>  kGcTypeSticky,
>
>  // Partial GC that marks the application heap but not the Zygote.
>
>  kGcTypePartial,
>
>  // Full GC that marks and frees in both the application and Zygote heap.
>
>  kGcTypeFull,
>
>  // Number of different GC types.
>
>  kGcTypeMax,



### 1. 区别

#### 1.1.1 Sticky GC

用途: 用于快速回收自上次 GC 以来创建并死亡的新生代对象。

特点: 非常高效，暂停时间极短。

**适用场景**：前台运行时使用，尤其是在短时间内创建和释放大量短命对象的情况下

#### 1.1.2 Partial GC

用途: 扫描和回收整个新生代，清理较多的短命对象。

特点: 暂停时间稍长于 Sticky GC，但依然适合前台操作。

**适用场景**：用于前台的常规内存管理，主要回收新生代对象。

#### 1.1.3 Full GC

用途: 虽然 Full GC 通常不会在前台主动触发，但在内存非常紧张且其他 GC 无法释放足够内存时，ART 可能会在前台触发 Full GC。

特点: 回收整个堆，包括新生代和老年代对象。暂停时间最长，对用户体验影响较大。



## 2. GC Cause

art/runtime/gc/gc\_cause.h

> enum GcCause {
>
>  // Invalid GC cause used as a placeholder.
>
>  kGcCauseNone,
>
>  // GC triggered by a failed allocation. Thread doing allocation is blocked waiting for GC before
>
>  // retrying allocation.
>
>  kGcCauseForAlloc,
>
>  // A background GC trying to ensure there is free memory ahead of allocations.
>
>  kGcCauseBackground,
>
>  // An explicit System.gc() call.
>
>  kGcCauseExplicit,
>
>  // GC triggered for a native allocation when NativeAllocationGcWatermark is exceeded.
>
>  // (This may be a blocking GC depending on whether we run a non-concurrent collector).
>
>  kGcCauseForNativeAlloc,
>
>  // GC triggered for a collector transition.
>
>  kGcCauseCollectorTransition,
>
>  // Not a real GC cause, used when we disable moving GC (currently for GetPrimitiveArrayCritical).
>
>  kGcCauseDisableMovingGc,
>
>  // Not a real GC cause, used when we trim the heap.
>
>  kGcCauseTrim,
>
>  // Not a real GC cause, used to implement exclusion between GC and instrumentation.
>
>  kGcCauseInstrumentation,
>
>  // Not a real GC cause, used to add or remove app image spaces.
>
>  kGcCauseAddRemoveAppImageSpace,
>
>  // Not a real GC cause, used to implement exclusion between GC and debugger.
>
>  kGcCauseDebugger,
>
>  // GC triggered for background transition when both foreground and background collector are CMS.
>
>  kGcCauseHomogeneousSpaceCompact,
>
>  // Class linker cause, used to guard filling art methods with special values.
>
>  kGcCauseClassLinker,
>
>  // Not a real GC cause, used to implement exclusion between code cache metadata and GC.
>
>  kGcCauseJitCodeCache,
>
>  // Not a real GC cause, used to add or remove system-weak holders.
>
>  kGcCauseAddRemoveSystemWeakHolder,
>
>  // Not a real GC cause, used to prevent hprof running in the middle of GC.
>
>  kGcCauseHprof,
>
>  // Not a real GC cause, used to prevent GetObjectsAllocated running in the middle of GC.
>
>  kGcCauseGetObjectsAllocated,
>
>  // GC cause for the profile saver.
>
>  kGcCauseProfileSaver,
>
>  // GC cause for deleting dex cache arrays at startup.
>
>  kGcCauseDeletingDexCacheArrays,
>
> };



> // 无效的 GC 原因用作占位符。
>
> kGcCauseNone,
>
> // GC 由失败的分配触发。执行分配的线程在重试分配之前被阻止等待 GC。
>
> kGcCauseForAlloc,
>
> // 后台 GC 试图确保在分配之前有可用内存。
>
> kGcCauseBackground,
>
> // 显式 System.gc() 调用。
>
> kGcCauseExplicit,
>
> // 当超过 NativeAllocationGcWatermark 时触发本机分配的 GC。
>
> //（这可能是一个阻塞 GC，具体取决于我们是否运行非并发收集器）。
>
> kGcCauseForNativeAlloc,
>
> // GC 因收集器转换而触发。
>
> kGcCauseCollectorTransition,
>
> // 不是真正的 GC 原因，当我们禁用移动 GC（当前用于 GetPrimitiveArrayCritical）时使用。
>
> kGcCauseDisableMovingGc,
>
> // 不是真正的 GC 原因，当我们修剪堆时使用。
>
> kGcCauseTrim,
>
> // 不是真正的 GC 原因，用于实现 GC 和插桩之间的排除。
>
> kGcCauseInstrumentation,
>
> // 不是真正的 GC 原因，用于添加或删除应用程序图像空间。
>
> kGcCauseAddRemoveAppImageSpace,
>
> // 不是真正的 GC 原因，用于实现 GC 和调试器之间的排除。
>
> kGcCauseDebugger,
>
> // 当前后台收集器都是 CMS 时触发后台转换的 GC。
>
> kGcCauseHomogeneousSpaceCompact,
>
> // 类链接器原因，用于保护使用特殊值填充艺术方法。
>
> kGcCauseClassLinker,
>
> // 不是真正的 GC 原因，用于实现代码缓存元数据和 GC 之间的排除。
>
> kGcCauseJitCodeCache,
>
> // 不是真正的 GC 原因，用于添加或删除系统弱持有者。
>
> kGcCauseAddRemoveSystemWeakHolder,
>
> // 不是真正的 GC 原因，用于防止 hprof 在 GC 中间运行。
>
> kGcCauseHprof,
>
> // 不是真正的 GC 原因，用于防止 GetObjectsAllocated 在 GC 中间运行。
>
> kGcCauseGetObjectsAllocated,
>
> // 配置文件保存程序的 GC 原因。
>
> kGcCauseProfileSaver,
>
> // 启动时删除 dex 缓存数组的 GC 原因。
>
> kGcCauseDeletingDexCacheArrays,





### 2.1 GC分类

GC指GC在GC回收线程(HeapTaskDaemon)执行，阻塞类GC在进程的工作线程执行。

###

#### 2.1.1 阻塞类

1. Alloc GC



* Explicit GC



#### 2.1.2 并发类

1. Background GC   目前遇到的GC大多都是这种类型。

> 后台GC，这里的“后台”并不是指应用切到后台才会执行的GC，而是GC在运行时基本不会影响其他线程的执行，所以也可以理解为并发GC。

* Native GC/NativeAlloc GC

> ART可以管理一些特定的Native内存。Bitmap使用NativeAllocationRegistry类申请和释放Native内存时会同步告知ART，ART可监控此类由java对象引用的Native对象内存。

* CollectorTransition GC



### 2.2 Background GC回收时机



## 3. Collector Type

art/runtime/gc/collector\_type.h

> // 未选择收集器。
>
> kCollectorTypeNone,
>
> // 非并发标记清除。
>
> kCollectorTypeMS,
>
> // 并发标记清除。
>
> kCollectorTypeCMS,
>
> // 并发标记压缩。
>
> kCollectorTypeCMC,
>
> // 并发标记压缩 GC 的后台压缩。
>
> kCollectorTypeCMCBackground,
>
> // 半空间/标记清除混合，启用压缩。
>
> kCollectorTypeSS,
>
> // 堆修剪收集器，不进行任何实际收集。
>
> kCollectorTypeHeapTrim,
>
> // (大多数) 并发复制收集器。
>
> kCollectorTypeCC,
>
> // 并发复制收集器的后台压缩。
>
> kCollectorTypeCCBackground,
>
> // 仪表临界区假收集器。
>
> kCollectorTypeInstrumentation,
>
> // 用于添加或删除应用程序映像空间的假收集器。
>
> kCollectorTypeAddRemoveAppImageSpace,
>
> // 用于实现 GC 和调试器之间排除的伪收集器。
>
> kCollectorTypeDebugger,
>
> // 当前台和后台收集器都是 CMS 时，在后台转换中使用的同质空间压缩收集器。
>
> kCollectorTypeHomogeneousSpaceCompact,
>
> // 类链接器伪收集器。
>
> kCollectorTypeClassLinker,
>
> // JIT 代码缓存伪收集器。
>
> kCollectorTypeJitCodeCache,
>
> // Hprof 伪收集器。
>
> kCollectorTypeHprof,
>
> // 用于安装/移除系统弱持有者的伪收集器。
>
> kCollectorTypeAddRemoveSystemWeakHolder,
>
> // GetObjectsAllocated 的伪收集器类型
>
> kCollectorTypeGetObjectsAllocated,
>
> // ScopedGCCriticalSection 的伪收集器类型
>
> kCollectorTypeCriticalSection,

http://aospxref.com/android-14.0.0\_r2/xref/art/runtime/runtime\_options.def#81



```c++
void Heap::ChangeCollector(CollectorType collector_type) {
  // TODO: Only do this with all mutators suspended to avoid races.
  if (collector_type != collector_type_) {
    collector_type_ = collector_type;
    gc_plan_.clear();
    switch (collector_type_) {
      case kCollectorTypeCC: {
        if (use_generational_cc_) {
          gc_plan_.push_back(collector::kGcTypeSticky);
        }
        gc_plan_.push_back(collector::kGcTypeFull);
        if (use_tlab_) {
          ChangeAllocator(kAllocatorTypeRegionTLAB);
        } else {
          ChangeAllocator(kAllocatorTypeRegion);
        }
        break;
      }
      case kCollectorTypeCMC: {
        gc_plan_.push_back(collector::kGcTypeFull);
        if (use_tlab_) {
          ChangeAllocator(kAllocatorTypeTLAB);
        } else {
          ChangeAllocator(kAllocatorTypeBumpPointer);
        }
        break;
      }
      case kCollectorTypeSS: {
        gc_plan_.push_back(collector::kGcTypeFull);
        if (use_tlab_) {
          ChangeAllocator(kAllocatorTypeTLAB);
        } else {
          ChangeAllocator(kAllocatorTypeBumpPointer);
        }
        break;
      }
      case kCollectorTypeMS: {
        gc_plan_.push_back(collector::kGcTypeSticky);
        gc_plan_.push_back(collector::kGcTypePartial);
        gc_plan_.push_back(collector::kGcTypeFull);
        ChangeAllocator(kUseRosAlloc ? kAllocatorTypeRosAlloc : kAllocatorTypeDlMalloc);
        break;
      }
      case kCollectorTypeCMS: {
        gc_plan_.push_back(collector::kGcTypeSticky);
        gc_plan_.push_back(collector::kGcTypePartial);
        gc_plan_.push_back(collector::kGcTypeFull);
        ChangeAllocator(kUseRosAlloc ? kAllocatorTypeRosAlloc : kAllocatorTypeDlMalloc);
        break;
      }
      default: {
        UNIMPLEMENTED(FATAL);
        UNREACHABLE();
      }
    }
    SetDefaultConcurrentStartBytesLocked();
  }
}
```



```c++
void Heap::ChangeAllocator(AllocatorType allocator) {
  if (current_allocator_ != allocator) {
    // These two allocators are only used internally and don't have any entrypoints.
    CHECK_NE(allocator, kAllocatorTypeLOS);
    CHECK_NE(allocator, kAllocatorTypeNonMoving);
    current_allocator_ = allocator;
    MutexLock mu(nullptr, *Locks::runtime_shutdown_lock_);
    SetQuickAllocEntryPointsAllocator(current_allocator_);
    Runtime::Current()->GetInstrumentation()->ResetQuickAllocEntryPoints();
  }
}
```







### 3.1 **TLAB**

* **定义**: TLAB (Thread-Local Allocation Buffer)  是Android Runtime (ART) 和其他垃圾回收系统（如 JVM）中使用的一种内存分配优化技术，专门用于提升对象分配的性能。TLAB为每个线程分配的一块私有内存区域，线程可以在这块区域内独立分配对象，而无需与其他线程竞争。这种方式避免了全局堆分配时的锁竞争，显著提高了多线程环境下的对象分配效率。

* **工作原理**:

  1. **线程私有内存**: 每个线程在堆内存中预分配一个小块内存作为 TLAB。线程可以直接在这块内存中分配对象，而不需要访问全局堆或与其他线程竞争。

  2. **分配对象**: 当线程需要分配一个新对象时，它首先尝试在自己的 TLAB 中分配。如果 TLAB 有足够的空间，分配操作非常快速，仅需移动一个指针即可。

  3. **TLAB 耗尽**: 如果 TLAB 中的内存不足以容纳新的对象，线程会请求分配一个新的 TLAB。此时，线程可能需要访问全局堆，导致短暂的锁竞争。

  4. **非 TLAB 分配**: 对于某些较大的对象，可能无法在 TLAB 中分配，它们将直接在全局堆中分配。

#### 3.1.1 **TLAB 的优点**

* **减少锁争用**: 通过给每个线程分配独立的内存区域，TLAB 减少了线程间的竞争，避免了多线程同时分配内存时的锁争用问题。

* **提升分配效率**: TLAB 内存分配是本地操作，通常只需要简单地移动指针，因此速度非常快。这种本地分配避免了全局堆内存管理带来的开销。

* **局部性优化**: TLAB 让每个线程在其局部内存中分配对象，减少了内存碎片化问题，并且提高了缓存命中率。

#### 3.1.2 **TLAB 的缺点**

* **内存浪费**: 由于每个线程都有一块预分配的内存区域，且每次线程请求的 TLAB 大小可能大于其实际需要，可能导致一些内存浪费，特别是在 TLAB 未使用完时线程终止的情况。

* **大对象分配**: 大对象无法在 TLAB 中分配，仍然需要访问全局堆，这可能带来一定的性能影响。

#### 3.1.3 **适用场景**

TLAB 适合于多线程高频率对象分配的场景，特别是应用中有大量小对象需要频繁分配时，使用 TLAB 可以显著提高性能。在 ART 中，TLAB 是分配器优化的一个重要部分，特别是在需要高并发性能的 Android 应用中，TLAB 有助于提升应用的响应速度和流畅性。

#### 3.1.4 **总结**

* **TLAB** 是一种面向线程的内存分配优化技术，能够显著提升多线程环境下对象分配的效率。

* 它通过为每个线程分配独立的内存区域，减少了全局堆内存分配时的锁竞争，适用于频繁对象分配的场景。

* 尽管存在一些内存浪费的可能性，但 TLAB 带来的分配速度提升往往能够抵消这些问题，尤其是在高并发的应用中。

## 4. Allocator Type

art/runtime/gc/allocator\_type.h

> // BumpPointer 空间目前仅用于 ZygoteSpace 构造。
>
> kAllocatorTypeBumpPointer，// 使用全局基于 CAS 的 BumpPointer 分配器。(\*)
>
> kAllocatorTypeTLAB，// 在 BumpPointer 空间内使用 TLAB 分配器。(\*)
>
> kAllocatorTypeRosAlloc，// 使用 RosAlloc（隔离大小、空闲列表）分配器。(\*)
>
> kAllocatorTypeDlMalloc，// 使用 dlmalloc（众所周知的 C malloc）分配器。(\*)
>
> kAllocatorTypeNonMoving，// 用于非移动对象的特殊分配器。
>
> kAllocatorTypeLOS，// 大型对象空间。
>
> // 以下内容与 BumpPointer 分配器的主要区别在于，内存是从多个区域分配的，而不是从单个连续空间分配的。
>
> kAllocatorTypeRegion，// 在区域内使用基于 CAS 的连续 bump-pointer 分配。 (\*)
>
> kAllocatorTypeRegionTLAB，// 使用区域片段作为 TLAB。大多数小对象的默认设置。(\*)



### 4.1 kUseReadBarrier

读屏障的实现有三种主要方式，分别适用于不同的场景和需求。

#### 4.1.1 **Baker's  Read Barrier**

* **概述**: 这是一种经典的读屏障实现方式，主要用于半空间复制（Semi-Space Copying）收集器，特别是在并发复制收集器（Concurrent Copying Collector）中广泛应用。

* **工作原理**: 当对象被访问（读取）时，Baker's Read Barrier 会检查该对象是否已经被复制到新空间。如果对象尚未被复制，它会立即将对象复制到目标空间（通常是 To-Space），并返回新地址。这种方式确保了并发垃圾回收期间访问的对象始终是最新的。

* **优点**:

  * 保证访问到的对象都是最新的（即已经复制到新空间的对象）。

  * 能够支持并发垃圾收集器，在不暂停应用程序的情况下进行对象复制。

* **缺点**:

  * 增加了每次对象访问的开销，因为每次读取对象时都需要执行读屏障逻辑。

#### 4.1.2 **Brooks's  Read Barrier**

* **概述**: Brooks's Read Barrier 是另一种读屏障实现方式，它与 Baker's Read Barrier 类似，但工作方式略有不同。Brooks's Algorithm 主要通过对象的自引用字段（Self-Referencing Pointer）来实现屏障逻辑。

* **工作原理**: 每个对象都有一个自引用字段，指向对象自身。当对象被复制到新空间时，自引用字段会更新为对象的新地址。每次访问对象时，Brooks's Read Barrier 检查并确保通过自引用字段访问最新的对象地址。

* **优点**:

  * Brooks's Read Barrier 可以简化对象复制的过程，因为自引用字段可以直接指向最新的对象地址。

  * 对象的访问路径一致性得以保持。

* **缺点**:

  * 每个对象需要额外的内存来存储自引用字段，增加了内存开销。

  * 读屏障的逻辑仍然会在每次对象访问时引入额外的计算。

#### 4.1.3 **Field Marking Read Barrier**

* **概述**: Field Marking 是一种较为复杂的读屏障实现方式，通常用于并发标记压缩（Concurrent Mark-Compact）或类似的垃圾收集器中。它通过对对象字段的标记来跟踪对象的状态，并在必要时进行更新或压缩。

* **工作原理**: 在并发标记阶段，Field Marking Read Barrier 会对对象的字段进行标记，以跟踪哪些字段已经被访问或更新。当字段被访问时，读屏障会检查该字段是否已标记，并执行相应的操作（如标记、更新或压缩）。

* **优点**:

  * Field Marking 允许垃圾收集器在不停止应用程序的情况下执行压缩和标记操作。

  * 适合需要对对象进行压缩或重新整理的垃圾收集器。

* **缺点**:

  * 实现复杂，读屏障逻辑引入的开销较大。

  * 在对象访问频繁的场景下，可能会导致性能下降。

#### 4.1.4 **总结**

* **Baker's Algorithm** 适用于并发复制收集器，确保对象在被访问时已复制到新空间。

* **Brooks's Algorithm** 通过自引用字段简化了对象复制，但需要额外的内存来存储自引用指针。

* **Field Marking Algorithm** 用于并发标记压缩收集器，通过标记字段来支持对象压缩和整理，适合更复杂的垃圾回收机制。

## 5. GC Collector对比

## 6. GC回收时机与水位计算

[Android R常见GC类型与问题案例-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/120574383)
