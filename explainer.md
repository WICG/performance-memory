# Objectives

## Goals

* Provide developers a mechanism to identify releases of their site that regress
  memory usage.

## Non-Goals

* Provide an incentive for developers to reduce memory footprint.
* Provide a mechanism to evalute tradeoffs between memory footprint and performance
  [e.g. caching]
* Provide per-iframe accounting for memory footprint.
* Provide a mechanism to perform in-depth memory footprint evaluations on a
  single running instance of a web page.
  * Memory footprint is dependent on garbage collection (GC) timings, which we
    do not wish to expose.
  * There is no simple API that can expose all relevant information for
    debugging memory issues.
* Provide a mechanism to compare memory usage between browser vendors.
  * The metrics in this API are implementation and context dependent.
    This invalidates comparisons between browser vendors.

# Problem statement

Some developers wish to reduce the memory footprint of their site, or at least
prevent it from getting worse. There is currently no API to measure the memory
footprint, thus making this task very difficult.

See [Appendix B](#appendix-b) for a description of Chrome's current
implementation of performance.memory, and why it does not solve the problem.

See [Appendix C](#appendix-c) for a list of problems found in real websites that
potentially could have been caught by this API.

# Use cases

Developers can collect the metric in aggregate, and perform staged rollouts to
see if the new version of the site regresses the metric.

To ensure that comparisons are being made in similar environments, we recommend
that developers aggregate the data across environments where the following
properties are identical:

* navigator.userAgent

# Proposed API

```
callback MemoryEstimateCallback = void (long? memoryEstimate);

partial interface Performance {
  // This callback provides an estimate of the memory footprint of the site, not
  // including memory associated with cross-origin iframes.

  // Memory estimate is only useful when collected in aggregate across a staged
  // rollout of a website. Comparing the distributions between old and new
  // versions of a website will provide insight into changes in memory
  // consumption caused by changes to the website itself.

  // The results from non-aggregated data [e.g. from individual calls to this
  // method] are not meaningful. Results may be time-quantized, or have
  // significant noise added to them.
  void getMemoryEstimate(MemoryEstimateCallback handler);
}
```

# Implementation

The semantics of memory usage are inherently tied to browser implementation
details. As such, the values returned by this API are not required to be
consistent when compared between browsers. This gives browser vendors a lot of
leeway with how to best implement the API.

## Guidelines

These guidelines are optional recommendations for browser vendors. For normative
requirements, see [Requirements](#requirements).

### What to include in memory estimate

The rule of thumb: memory estimate should reflect memory that is directly
attributable to web developers of the site.

After all, the goal is to help web developers find changes in their code that
unintentionally increase memory usage.

Examples of memory that should be included:
* DOM elements and associated backing stores.
* All memory needed to host script and CSS for the site.

Examples of memory that should be excluded:
* Graphics contexts used to render and display the site.
  * The magnitude of this memory is frequently proportional to the size of the
    window and/or display device, which are outside of the control of the
    developer.
  * The existence of these resources is frequently correlated with whether the
    site is in a foreground or background tab, which is also outside of the
    control of the developer.

Examples of memory whose inclusion is unclear:
* Memory needed to host third-party JS libraries used by the site.
  * Excluding this would be difficult for browser vendors to implement.
* Memory needed to host iframes used by the site.
* Shared memory. See [Appendix A](#appendix-a)

### Context free

As much as possible, memory estimate should not reflect context outside of the
control of the web developer. For example, memory estimate should not reflect:
* The size of the window
* The size or resolution of the display device
* Whether the operating system is experiencing memory pressure
  * Modern OSes will compress memory and/or swap it out to disk under memory
    pressure. This can drastically affect the memory usage of the browser.
* Whether the tab is visible
* Memory that can be reused/discarded by the operating system at any point in
  time.
  * madvise (Linux variants + macOS), ashmem (Android), VirtualAllocEx (Windows)
    are examples of APIs that can be used to lazily free memory. Exact semantics
    are dependent on the operating system. Browser vendors are advised to tread
    with caution.
* File-backed memory.
  * Similar to reusable/discardable memory, operating systems will clean and
    discard pages from file-backed memory under memory pressure.
  * Typical sources of file-backed memory include the browser binary, system
    libraries and system resources, all of which are outside of the control of
    the web developer.

## Requirements

### Returning null

Browsers are always allowed to return null in place of a memory estimate. A
browser which does not want web developers to optimize for this metric could
choose to always return null.

Browsers must return null if they are unable to compute a memory estimate that is
isolated to the site calling the API.

During a browing session for a site, browsers are allowed to return both null
and non-null values for separate calls to getMemoryEstimate.

### Comprehensive measurements

Web developers will be optimizing their websites to minimize the number returned
by getMemoryEstimate. Browser vendors must return comprehensive measurements.

For example, if the implementation of getMemoryEstimate failed to include the
backing store for Canvas2D elements, this would pressure web developers to use
more Canvas2D elements, believing that they were improving the memory footprint
of their site.

At a minimum, getMemoryEstimate must return memory used for:
* All DOM nodes, including backing store for Canvas2D, image, video, SVG, and
  WebGL elements.
* All CSS.
* All javascript used by same-origin iframes from the site, including array
  buffers and web assembly.

## Security

A malicious attacker could frequently poll this API to attempt to get
information about garbage collection timings or other implementation details
inadvertently exposed by the API.

This specification recommends [but does not require]:
* Quantizing measurements in the time domain with a threshold of at least 30
  seconds.
  * By only returning new measurements every 30 seconds, this restricts
    information available to the attacker down to: There was a GC sweep in the
    last thirty seconds.
* Adding normalized noise to each measurement.
  * Since the API is intended to be used in aggregate, the normalized noise will
    not affect the mean of the distribution, but will greatly reduce precision
    of information available to attackers.

## Privacy

There are three data points exposed by this API which could be used to garner
additional information about the user.
* Whether or not the API returns null.
* If the successive calls to the API transition from null to non-null or from
  non-null to null.
* Substantial variations in the returned value could indicate major changes in
  browing state.
  * e.g. If a site (origin A), adds an iframe to (origin B), it is possible that
    the API could be used to get information about whether the iframe (origin B)
    itself includes an iframe from (origin A).

The privacy implications of these data points will be dependent on the
implementation of the API. Browser vendors are recommended to perform their own
analysis.

# <a name="appendix-a"></a> Appendix A - Shared Memory

The semantics of accounting for shared memory is poorly defined. If a memory
region is mapped into multiple processes [possibly multiple times], and is being
used by multiple sites, which ones should it count towards?

On Linux, one common solution is to use proportional set size, which counts
1/Nth of the resident size, where N is the number of other mappings of the
region. This has the nice property of being additive across processes. The
downside is that it is context dependent. e.g. If a user opens more tabs, thus
causing a system library to be mapped into more processes, the PSS for previous
tabs will go down.

This specification recommends that all shared memory be attributed a single
owner. It should count with its full weight in any memory estimates for that
owner, and count as nothing for all other memory estimates. This retains the
additive property, while still remaining context free.

# <a name="appendix-b"></a> Appendix B - Chrome's Previous Implementation of performance.memory

The current implementation of performance.memory in Chrome exposes
usedJSHeapSize and totalJSHeapSize. These metrics have problems:
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
* The numbers are quantized into buckets and time-delayed.

Moreover performance.memory in Chrome exposes jsHeapSizeLimit which is an JavaScript engine specific
heuristic which may not exisit in other JavaScript engines.

# <a name="appendix-c"></a> Appendix C - Problems Affecting Real Websites

This section contains a list of memory-related problems discovered on real
websites. These problems could have potentially been caught by the web
developers if they were using this API when rolling out the changes that
introduced the issues. The issues ranged in size from a couple of MB to 1GB+.

* Every five minutes, a script would add [but never remove] an event listener
  for window.unload. The callback was an anonymous function that inadvertently
  captured a significant amount of context.
  * This pattern was observed on multiple occasions.
* A site was storing error messages without bound. These included stringified,
  large JSON objects.
* A site was creating a large number of duplicates of semantically immutable
  data.
  * This pattern was observed on multiple occasions.
* A site was creating at startup a large array buffer that would usually remain
  untouched for the duration of the session [rarely used feature].
* A site was unnecessarily creating hundreds of Canvas2D elements.

# Appendix D - Possible Future Extensions

It would be helpful to provide web developers more categories of memory usage,
to help narrow down sources of regressions. Potential categories include JS
memory usage, DOM nodes, CSS, Canvas, audio/video, etc. While Chrome does have
some internal accounting for this, coming up with consistent, low-overhead,
cross-browser definitions seems difficult. We may wish to consider adding an
"implementation profiling hints" dictionary which vendors can use to expose
vendor-specific measurements.

