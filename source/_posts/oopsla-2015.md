---
title: "[OOPSLA '15] Accurate Profiling in the Presence of Dynamic Compilation"
date: 2015-10-23 00:00:00
comments: false
tags: publication
---

by **Yudi Zheng**, Lubomir Bulej, and Walter Binder.

In this paper, we present a technique to make bytecode-instrumentation-based profiler aware of optimizations performed by the dynamic compiler. We implement our approach in Graal and demonstrate its significance with concrete profilers. In particular, we quantify the impact of escape analysis on allocation profiling, object life-time analysis, and the impact of method inlining on callsite profiling.

Download at [[link]][1].

[1]: https://doi.org/10.1145/2814270.2814281
