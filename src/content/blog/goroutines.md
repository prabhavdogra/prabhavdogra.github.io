---
title: "Writing your own Goroutines"
date: 2025-03-01
description: "An in-depth look at Go’s sync.Once: its design, internal implementation, and how it guarantees that a function runs exactly once, even across concurrent goroutines."
---

This all started when someone asked me how goroutines work internally and all I could respond with was:

> "Goroutines are lightweight threads managed by the Go runtime instead of the operating system. Go runtime automatically multiplexes—mapping multiple goroutines onto a smaller number of OS threads. And that somehow makes them fast?? 👉👈"

If anyone asked me any in-depth questions about how this multiplexing worked I was blank. So I decided to gain a deeper understanding by implementing goroutines myself. Cloned the [Go Github repo](https://github.com/golang/go)

## To be continued...

### Go runtime:
<details>
  <summary>runtime/proc.go</summary>

    - It handles the core scheduling and lifecycle of Go programs.
</details>

<details>
  <summary>How memory allocation works in Golang?</summary>
- Entrypoint: 
    - `runtime/proc.go : schedinit()` → `runtime/malloc.go : mallocinit()`

        - `mallocinit()`: It performs a series of checks and sets up the fundamental data structures and configurations necessary for dynamic memory allocation within your Go program.
            - Invokes `mheap.init()`

    - `runtime/mheap.go`
        - asdf
    - `runtime/mcache.go`
    - `runtime/mcentral.go`
    - `runtime/msize.go`
    - `runtime/mstats.go`
    - `runtime/mgc.go`
    - `runtime/malloc.go`
</details>


### References
- Goroutines and their scheduling basics:
    - https://www.youtube.com/watch?v=S-MaTH8WpOM [Goroutines: Vicki Niu]
    - https://youtu.be/MYtUOOizITs?si=FVGFtez2z3fNCjx7 [Goroutines: jesus espino]
    - https://youtu.be/wQpC99Xu1U4?si=uOu0RiLyMpNXKYa0 [Go Scheduler: Madhav Jivrajani]
- Go scheduler basics
    - Channel primitives: https://youtu.be/KBZlN0izeiY?si=8HAeSVJxE3Vc3GC0 [Channels - Kavya Joshi]
- How memory allocation works in go
    - https://goog-perftools.sourceforge.net/doc/tcmalloc.html [tcmalloc]
    - https://andrestc.com/post/go-memory-allocation-pt1/ [tcmalloc inspired allocator]
- Go garbage collection internals:
    - https://youtu.be/gPxFOMuhnUU?si=O9pn99sLiqptgyw3 [Garbage collector: Maya Rosecrance]
    - https://youtu.be/We-8RSk4eZA?si=QNXxqq2xVEoh9At9 [GC Pacer: Madhav Jivrajani]
- Netpoll: https://youtu.be/xwlo3xigknI?si=dmTrK_CH_fa0Bs51 [netpoll - Cindy Sridharan]
