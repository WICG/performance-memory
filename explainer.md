# Objectives

## Goals

* Provide developers an intuitive memory metric that can be used to identify
  releases that unintentionally increase memory footprint.

## Non-Goals

* Provide an incentive for developers to reduce memory footprint.
* Provide a mechanism to evalute tradeoffs between memory footprint and performance
  [e.g. caching]
* Provide per-frame accounting for memory footprint.
* Provide a mechanism to perform in-depth memory footprint evaluations on a
  single running instance of a web page.
  * Memory footprint is dependent on garbage collection (GC) timings, which we
    have no desire to expose.
  * There is no simple API that can expose all relevant information for
    debugging memory issues.

# Problem statement

Some developers wish to reduce the memory footprint of their site, or at least
prevent it from getting worse. There is currently no API to measure the memory
footprint, thus making this task very difficult.

See [Appendix D](#appendix-d) for a description of Chrome's current
implementation of performance.memory, and why it does not solve the problem.

# Use cases

Developers can collect the metric in aggregate, and perform staged rollouts to
see if the new version of the site regresses the metric.

# Requirements

* Context free
  * Actions that a user takes on other sites or on other applications should not
    affect the memory footprint of the site calling the API.
* Minimal/no overhead when the API is not used.
* Comprehensive
  * Should account for all memory allocated on behalf of the web page, including
    memory required by the browser's internal structures.
* Actionable
  * When used as a signal in aggregate, there should be low false positives and
    low false negatives.
  * An unexpected regression in the aggregate number implies that there is a high
    probability that there is an unintentional memory-related coding error.
  * With high probability, memory-related coding errors should cause regression
    in the aggregate metric.
  * The usedJSHeapSize and totalJSHeapSize sub-metrics provides some insight
    into the possible source of bloat.
* Accurate or null
  * If accurate metrics cannot be obtained, return null. This is better than
    returning inaccurate numbers. See [Appendix C](#appendix-c) for more
    details.
* Definition consistent on all [supported platforms](#supported-platforms).

# Proposed API

```
partial interface Performance {
  [SameObject] readonly attribute MemoryInfo memory;
}

// All values will be |null| if there are multiple top-level frames being hosted
// by the same process. Once site-isolation is enabled, this will never be the
// case.
interface MemoryInfo {
  // All private memory being used by the process hosting the site.
  readonly attribute unsigned long? privateMemoryFootprint;

  // Sum of size of all JS-related entities in the process hosting the site.
  // Includes objects, functions, closures, array buffers, etc.
  readonly attribute unsigned long? usedJSHeapSize;

  // All memory used by the JS heap in the process hosting the site.
  // Includes everything from usedJSHeapSize, and also fragmentation.
  readonly attribute unsigned long? totalJSHeapSize;
}
```

# Proposed Implementation Outline

For **privateMemoryFootprint**, **totalJSHeapSize**, and **usedJSHeapSize** we
only have accurate accounting when the process is hosting a single top level
frame. As such, we return null anytime the process is hosting more than a single
top level frame.

We define **privateMemoryFootprint** as non-reusable, private, anonymous,
resident/swapped/compressed memory. See [Appendix A](#appendix-a) for
definitions of these terms.

**usedJSHeapSize** reflects the accumulative size of objects memory used by the JS implementation for a given
execution context (main thread or worker), including objects, functions, closures, array buffers, etc. It does
not include memory of objects used by the DOM, the browser vendor's internal data structures, and memory used
by graphics/audio libraries.

**totalJSHeapSize** reflects memory used by the JS implementation heap to store JS objects for a given
execution context (main thread or worker). This includes objects, functions, closures, array buffers, etc.
and free memory (fragmentation) in between these objects that cannot be used for anything else than other
JS objects. It does not include memory used by the DOM, the browser vendor's internal data structures, and
memory used by graphics/audio libraries. Note that totalJSHeapSize will always be larger or equal than usedJSHeapSize.

Both **totalJSHeapSize** and **usedJSHeapSize** correspond to the execution context (main thread or worker)
where the call is performed.

## Rationale

* **non-reusable**: On macOS, the [implementation of
  libMalloc](https://opensource.apple.com/source/libmalloc/libmalloc-140.40.1/src/nano_malloc.c.auto.html)
  will sometimes call ```madvise(..., MADV_FREE_REUSABLE)``` instead of calling
  mach_vm_deallocate on pages. These pages are, for all intents and purposes,
  freed, despite still being potentially resident.
* **anonymous**: The primary use of file-backed pages [non-anonymous] is for
  loading the browser binary, system libraries, and other data files [e.g.
  fonts, localization, etc.]. These memory regions introduce cross-platform
  differences which are mostly outside of the control of site developers.
* **private**: The semantics of accounting for shared memory are unclear. See
  [Appendix B](#appendix-b) for more details. Furthermore, the site rarely has
  direct control over the use of shared memory - it's primarily used as an
  optimization by browser vendors. Not including shared memory in the
  calculation avoids all of these complications.
* **resident/swapped/compressed**: Whether memory is resident, swapped, or
  compressed is context dependent - it primarily depends on system memory
  pressure, and whether the memory was recently used. For compressed memory, we
  count pre-compression size to avoid variance in compression ratio, which depends
  on contents of compressed content.

## Pseudo-code

```
def GetPrivateMemoryFootprint:
  if process.HostsMoreThanOneTopLevelFrame():
    return null
  switch(platform):
    case Windows:
      return PROCESS_MEMORY_COUNTERS_EX.PrivateUsage
    case Darwin:
      task_info = task_info(TASK_VM_INFO)
      if (Darwin.supports_phys_footprint):
        return task_info.phys_footprint - GetAnonymousResidentSharedMemory()
      return task_info.internal + task_info.compressed - GetAnonymousResidentSharedMemory()
    default [Linux-derivative]:
      status = /proc/<pid>/status
      return status.RssAnon + status.VmSwap

def GetAnonymousResidentSharedMemory:
  <Requires the browser vendor to internally account for anonymous, resident,
  shared memory regions on macOS>.

def GetTotalJSHeapSize:
  <Requires the browser vendor to internally account for JS heap usage>

def GetUsedJSHeapSize:
  <Requires the browser vendor to internally account for JS heap usage>
```

## Drawbacks
In order to keep the implementation of GetPrivateMemoryFootprint fast, we use
OS-specific per-process counters. This means that each calculation is just a
single syscall, but we will see platform-specific differences in the per-process
counters.

* On Windows, PROCESS_MEMORY_COUNTERS_EX.PagefileUsage overcounts by committed
  memory that is not in the working set, or paged or compressed. This
  overcounting primarily stems from a fundamental difference in memory model
  between Linux and Windows. On Windows, committed memory counts towards the
  total system commit charge, and thus is the relevant memory metric to track.
* On macOS, task_info.compressed is comparable to VmSwap on Linux.
  task_info.internal is similar to RssAnon, but it also includes faulted,
  anonymous shared memory. We discount for this by keeping track of anonyous
  shared memory, and then computing resident, anonmous shared memory on demand.
  This undercounts, because it's possible for pages in a shared memory region to
  be resident, but not faulted.
* On macOS, phys_footprint is similar to internal + compressed, but also
  includes IOKit memory. While the memory is technically shared rather than
  private, functionally it behaves similarly to private memory. Thus, counting
  it as part of privateMemoryFootprint seems appropriate.

# <a name="appendix-a"></a>Appendix A - Terminology

Each platform exposes a different memory model. This section describes a
consistent set of terminology that will be used by this document. This
terminology is intentionally Linux-biased, since that is the platform most
readers are expected to be familiar with.

## <a name="supported-platforms"></a>Supported platforms
* Linux
* Android
* ChromeOS
* Windows [kernel: Windows NT]
* macOS/iOS [kernel: Darwin/XNU/Mach]

## Terminology
Warning: This terminology is neither complete, nor precise, when compared to the
terminology used by any specific platform. Any in-depth discussion should occur
on a per-platform basis, and use terminology specific to that platform.

* **Virtual memory** - A per-process abstraction layer exposed by the kernel. A
  contiguous region divided into 4kb **virtual pages**.
* **Physical memory** - A per-machine abstraction layer internal to the kernel.
  A contiguous region divided into 4kb **physical pages**. Each **physical
  page** represents 4kb of physical memory.
* **Resident** - A virtual page whose contents is backed by a physical
  page.
* **Swapped/Compressed** - A virtual page whose contents is backed by
  something other than a physical page.
* **Swapping/Compression** - [verb] The process of taking Resident pages and
  making them Swapped/Compressed pages. This frees up physical pages.
* **Unlocked Discardable/Reusable** - Android [Ashmem] and Darwin specific. A virtual
  page whose contents is backed by a physical page, but the Kernel is free
  to reuse the physical page at any point in time.
* **Private** - A virtual page whose contents will only be modifiable by the
  current process.
* **Copy on Write** - A private virtual page owned by the parent process.
  When either the parent or child process attempts to make a modification, the
  child is given a private copy of the page.
* **Shared** - A virtual page whose contents could be shared with other
  processes.
* **File-backed** - A virtual page whose contents reflect those of a
  file.
* **Anonymous** - A virtual page that is not file-backed.

## Platform Specific Sources of Truth
Memory is a complex topic, fraught with potential miscommunications. In an
attempt to forestall disagreement over semantics, these are the sources of truth
used by the authors to determine memory usage for a given process.

* Windows: [SysInternals
  VMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/vmmap)
* Darwin:
  [vmmap](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/vmmap.1.html)
* Linux/Derivatives:
  [/proc/\<pid\>/smaps](http://man7.org/linux/man-pages/man5/proc.5.html)

# <a name="appendix-b"></a> Appendix B - Shared Memory

Accounting for shared memory is poorly defined. If a memory region is mapped
into multiple processes [possibly multiple times], which ones should it count
towards?

On Linux, one common solution is to use proportional set size, which counts
1/Nth of the resident size, where N is the number of other mappings of the
region. This has the nice property of being additive across processes. The
downside is that it is context dependent. e.g. If a user opens more tabs, thus
causing a system library to be mapped into more processes, the PSS for previous
tabs will go down.

File backed shared memory regions are typically not interesting to report, since
they typically represent shared system resources, libraries, and the browser
binary itself, all of which are outside of the control of developers.

In Chrome, we have implemented ownership tracking for anonymous shared memory
regions - each shared memory region counts towards exactly one process, which is
determined by the type and usage of the shared memory region. We considered
exposing these numbers in performance.memory, but it seemed like it would cause
more confusion than it would help. The two main use cases are shared tiles
between a renderer and GPU process, and shared network resources between a
renderer and the browser process. Both of these are Chrome-specific
optimizations, context-dependent, and pretty much entirely outside of the
control of developers.

# <a name="appendix-c"></a> Appendix C - Time-Delay, Bucketing, Site-Isolation

The current implementation of performance.memory in Chrome applies quantization
and bucketing to the returned numbers. There is a [blink intent to
implement](https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/no00RdMnGio)
which desribes updating performance.memory to return null if there are multiple
top level frames being hosted in a process. If the site is isolated, then the
time-delay and bucketing are removed.

# <a name="appendix-d"></a> Appendix D - Chrome's Previous Implementation of performance.memory

The current implementation of performance.memory in Chrome exposes
usedJSHeapSize and totalJSHeapSize. These metrics have two problems:
* They cannot be used to identify regressions because they only represents part of renderer memory usage.
  * In the wild, totalJSHeapSize typically ranges from 25-40% of renderer memory
usage.
  * They does not include DOM memory [Oilpan, PartitionAlloc], canvas backing stores,
decoded images/videos, etc.
  * It is possible for developers to improve both usedJSHeapSize and totalJSHeapSize by shifting memory into
other allocators.
* It is non-intuitive because it only includes part of the memory developers
typically associate with JS.
  * Large ArrayBuffers are not included in usedJSHeapSize and totalJSHeapSize,
    since they are backed by PartitionAlloc.
  * External strings are not included in usedJSHeapSize and totalJSHeapSize
  
Moreover performance.memory in Chrome exposes jsHeapSizeLimit which is an JavaScript engine specific
heuristic which may not exisit in other JavaScript engines. Therefore, we propose to remove this metric.
