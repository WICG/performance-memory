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

See [Appendix A](#appendix-a) for a description of Chrome's current
implementation of performance.memory, and why it does not solve the problem.

See [Appendix B](#appendix-b) for a list of problems found in real websites that
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
partial interface Performance {
  // This callback provides an estimate of the memory footprint of the site.

  // Memory estimate is only useful when collected in aggregate across a staged
  // rollout of a website. Comparing the distributions between old and new
  // versions of a website will provide insight into changes in memory
  // consumption caused by changes to the website itself.

  // The results from non-aggregated data [e.g. from individual calls to this
  // method] are not meaningful. Results may be time-quantized, or have
  // significant noise added to them.

  // A null value means that the browser was unable to compute a memory estimate.
  // Note: Noise added by the browser may cause the result to be negative.
  Promise<long?> getMemoryEstimate();
}
```

# Implementation

The semantics of memory usage are inherently tied to browser implementation
details. As such, the values returned by this API are not required to be
consistent when compared between browsers. This gives browser vendors a lot of
leeway with how to best implement the API.

## Guidelines

These guidelines provide a framework to help browser vendors decide which memory
should be included in memory estimate. For normative requirements, see
[Requirements](#requirements).

* Memory estimate should be directly attributable to the web developer.
  * The goal is to help web developers find changes in *their* code that
    increase memory usage.
* Memory estimate should be context free.
  * It should not reflect memory outside of the control of the web developer.
  * See [Appendix C](#appendix-c) for examples of context-dependent memory.

## Requirements

### Returning null

Browsers are always allowed to return null in place of a memory estimate. A
browser which does not want web developers to optimize for this metric could
choose to always return null.

Browsers must return null if they are unable to compute an accurate memory
estimate.

During a browing session for a site, browsers are allowed to return both null
and non-null values for separate calls to getMemoryEstimate.

### Comprehensive measurements

Web developers will be optimizing their websites to minimize the number returned
by getMemoryEstimate. To prevent improper optimizations, browser vendors must
return comprehensive measurements.

For example, if the implementation of getMemoryEstimate reported memory usage
for small strings but not large strings, this would pressure web developers to
use larger strings, potentially even concatenating small strings into large
strings, believing that they were improving the memory footprint of their site.

At a minimum, getMemoryEstimate must include memory used by the current browsing
context for:
* All DOM nodes, including backing store for Canvas2D, image, video, SVG, and
  WebGL elements.
* The internal representation for all
  [resources](https://html.spec.whatwg.org/#resources). This may include
  compressed images, style sheets, and the source itself.
* Plugins.
* All script, including web assembly and Web Workers. This includes the backing
  store for array buffers.

### Related Similar-Origin Browsing Contexts

Browsing contexts can keep alive resources in [related similar-origin browsing
contexts](https://html.spec.whatwg.org/multipage/browsers.html#groupings-of-browsing-contexts).
Since JavaScript is a garbage-collected language, there's no way to attribute
ownership of a subset of resources in a browsing context. As such, memory
estimate must include all resources in related similar-origin browsing contexts.

See the first example in [Appendix B](#appendix-b) for an example of a memory
problem involving retention of resources across browsing contexts.

Aside: It's possible, given a resource X, to compute a graph of all resources
retained by X. This could be used to provide a maximal retainable-subset of
resources to use in different browsing contexts. This has two problems:
* This will be slow.
* This is context dependent, and potentially has high variance in time - small
  JavaScript changes [e.g. creation of a JavaScript function which captures a
  global variable] could lead to drastic changes in the maximal
  retainable-subset of resources.

The following three statements are equivalent:

* Memory estimate must include all memory retainable by JavaScript execution
  contexts that can potentially (via document.domain) be synchronously scripted
  from the current JavaScript execution context.
  * This includes memory that is potentially [but not currently] retained by the
    JavaScript execution context.
* Memory estimate must include all memory retainable by JavaScript execution
  contexts in this [unit of related similar-origin browsing
  contexts](https://html.spec.whatwg.org/multipage/browsers.html#groupings-of-browsing-contexts).
* Memory estimate must include:
  * All similar-origin iframes in the current window,
  * All similar-origin iframes in windows opened via window.open (without "noopener")
  * All similar-origin iframes in windows opened via links with target="_blank"
    (without "noopener").
  * Two origins are similar if they share the same eTLD+1.

Memory estimate should always exclude memory from different-origin (non-similar)
iframes.

See [Appendix D](#appendix-d) for examples of which browsing contexts to include
in the memory estimate.

### Shared Resources

Some resources can be shared by multiple browsing contexts. If the resource is
used by any related similar-origin browsing context, it must be included in the
memory estimate.

getMemoryEstimate must include memory used for:
* ServiceWorkers controlling any window, iframe or worker that is otherwise
  being included in the memory estimate.
* SharedWorkers accessible by any window, iframe or worker that is otherwise
  being included in the memory estimate.
* Shared array buffers used by any JavaScript execution context that is
  otherwise being included in the memory estimate.

### Disclaimer

Differences between browser vendor implementations of getMemoryEstimate are
expected since memory is an implementation detail. The exact behavior is less
important, as long as the implementation:
* Is internally consistent.
* Is comprehensive.
* Helps web developers find memory issues.
* Protects the user's privacy and security.

## Security

A malicious attacker could frequently poll this API to attempt to get
information about garbage collection timings or other implementation details
inadvertently exposed by the API.

This specification recommends [but does not require]:
* Quantizing measurements in the time domain with a threshold of at least 30
  seconds.
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
implementation of the API. Browser vendors should perform their own analysis.

# <a name="appendix-a"></a> Appendix A - Chrome's Current Implementation of performance.memory

The current implementation of performance.memory in Chrome exposes
usedJSHeapSize and totalJSHeapSize. These metrics have problems:
* They cannot be used to identify regressions because they only represents part of renderer memory usage.
  * In the wild, totalJSHeapSize typically ranges from 25-40% of renderer memory
usage.
  * They does not include DOM memory [Oilpan, PartitionAlloc], canvas backing stores,
decoded images/videos, etc.
  * It is possible for developers to improve both usedJSHeapSize and totalJSHeapSize by shifting memory into
other allocators.
* The numbers are quantized into 5MB buckets. This does not provide developers
  enough granularity to determine releases that regress memory.
  * See feedback from [Facebook](https://github.com/WICG/performance-memory/issues/7) and [GSuite and Gmail](https://github.com/WICG/performance-memory/issues/8).

# <a name="appendix-b"></a> Appendix B - Problems Affecting Real Websites

This section contains a list of memory-related problems discovered on real
websites. These problems could have potentially been caught by the web
developers if they were using this API when rolling out the changes that
introduced the issues. The issues ranged in size from a couple of MB to 1GB+.

* Every five minutes, a script would add [but never remove] an event listener
  for window.unload. The callback was an anonymous function that inadvertently
  captured a significant amount of context from a same-origin iframe. The iframe
  was subsequently removed from the DOM, but the event listener kept alive much
  of the memory.
  * This pattern was observed on multiple occasions.
* A site was storing error messages without bound. These included stringified,
  large JSON objects.
* A site was creating a large number of duplicates of semantically immutable
  data.
  * This pattern was observed on multiple occasions.
* A site was creating at startup a large array buffer that would usually remain
  untouched for the duration of the session [rarely used feature].
* A site was unnecessarily creating hundreds of Canvas2D elements.

# <a name="appendix-c"></a> Appendix C - Examples of context-dependent memory

* Memory used by graphics contexts to display the site.
  * The magnitude is typically proportional to the size of the window.
  * The magnitude is typically proportional to the scale factor of the display.
  * The magnitude is dependent on whether the window is visible.
* Memory that can be reused/discarded by the operating system at any point in
  time.
  * madvise (Linux variants + macOS), ashmem (Android), VirtualAllocEx (Windows)
    are examples of APIs that can be used to lazily free memory. Exact semantics
    are dependent on the operating system.
* File-backed memory.
  * Similar to reusable/discardable memory, operating systems will clean and
    discard pages from file-backed memory under memory pressure.
  * Typical sources of file-backed memory include the browser binary, system
    libraries and system resources, all of which are outside of the control of
    the web developer.

# <a name="appendix-d"></a> Appendix D - Examples of memory estimate

Each box with a URL represents a browsing context. Arrows depict the
relationship between them. In each example, the leftmost browsing context calls
getMemoryEstimate(). The color of each browsing context depicts whether it
should be included in the memory estimate.

![Example of memory estimate](/example.png)
