This is a living document and at times it will be out of date. It is
intended to articulate how programming in the Go runtime differs from
writing normal Go. It focuses on pervasive concepts rather than
details of particular interfaces.

Scheduler structures
====================

The scheduler manages three types of resources that pervade(遍布) the
runtime: Gs, Ms, and Ps. It's important to understand these even if
you're not working on the scheduler.

Gs, Ms, Ps
----------

A "G" is simply a goroutine. It's represented by type `g`. When a
goroutine exits, its `g` object is returned to a pool of free `g`s and
can later be reused for some other goroutine.
一个G代表一个goroutine, 使用类型g表示, 当一个goroutine退出, 其g对象返回
free g列表，并且之后可重用. 推论1: goroutine新建和毁灭的开销低

An "M" is an OS thread that can be executing user Go code, runtime
code, a system call, or be idle. It's represented by type `m`. There
can be any number of Ms at a time since any number of threads may be
blocked in system calls.
M代表一个OS thread, 有几种状态: 执行Go代码, 执行runtime代码, 系统调用或者空闲
任意时间可以有任意多的M, 因为可以有任意多个线程阻塞在系统调用

Finally, a "P" represents the resources required to execute user Go
code, such as scheduler and memory allocator state. It's represented
by type `p`. There are exactly `GOMAXPROCS` Ps. A P can be thought of
like a CPU in the OS scheduler and the contents of the `p` type like
per-CPU state. This is a good place to put state that needs to be
sharded for efficiency, but doesn't need to be per-thread or
per-goroutine.
P代表需要执行Go代码的资源, 比如调度器和内存分配状态. 恰好有GOMAXPROCS个P

The scheduler's job is to match up a G (the code to execute), an M
(where to execute it), and a P (the rights and resources to execute
it). When an M stops executing user Go code, for example by entering a
system call, it returns its P to the idle P pool. In order to resume
executing user Go code, for example on return from a system call, it
must acquire a P from the idle pool.
调度器的任务是配对G, M和P. 当M停止执行用户Go代码(比如系统调用阻塞), 其会将其P返回
闲置的P池, 使得这个P能被其他M获得, 这个P上的其他goroutine可以被执行. 要恢复执行
用户Go代码, 比如从系统调用返回, 其必须从空闲P池中获取一个P.

All `g`, `m`, and `p` objects are heap allocated, but are never freed,
so their memory remains type stable. As a result, the runtime can
avoid write barriers in the depths of the scheduler.
所有的`g`, `m`, `p`对象都是在堆上分配的, 永远不会被释放, 所以其内存是稳定的.
这导致runtime可以在调度器的层级避免写屏障

User stacks and system stacks
-----------------------------

Every non-dead G has a *user stack* associated with it, which is what
user Go code executes on. User stacks start small (e.g., 2K) and grow
or shrink dynamically.
每个未死亡的G都有一个用户栈, 用户Go代码在其上执行. 用户栈开始时从2K开始动态增长和缩小

Every M has a *system stack* associated with it (also known as the M's
"g0" stack because it's implemented as a stub G) and, on Unix
platforms, a *signal stack* (also known as the M's "gsignal" stack).
System and signal stacks cannot grow, but are large enough to execute
runtime and cgo code (8K in a pure Go binary; system-allocated in a
cgo binary).
每个M有个系统栈(也被称为g0栈因为其以一个stub G实现)和一个信号栈(Unix平台下).
这两个栈不能增长, 但已经足够大到执行运行时和cgo代码(纯Go二进制有8K, cgo二进制里是系统分配的)

Runtime code often temporarily switches to the system stack using
`systemstack`, `mcall`, or `asmcgocall` to perform tasks that must not
be preempted, that must not grow the user stack, or that switch user
goroutines. Code running on the system stack is implicitly
non-preemptible and the garbage collector does not scan system stacks.
While running on the system stack, the current user stack is not used
for execution.
runtime代码经常切换到系统栈执行不可被抢占的任务，不能增长用户栈的任务或切换用户goroutine的任务(?),
使用`systemstack`, `mcall`, `asmcgocall`.
运行在系统栈上的代码隐式不可被抢占，并且垃圾回收不扫描系统栈.
当运行在系统栈时, 当前用户栈不用来执行.

`getg()` and `getg().m.curg`
----------------------------

To get the current user `g`, use `getg().m.curg`.
要获取当前用户`g`, 使用`getg().m.curg`.

`getg()` alone returns the current `g`, but when executing on the
system or signal stacks, this will return the current M's "g0" or
"gsignal", respectively. This is usually not what you want.
单独的`getg()`会返回当前`g`, 但当在系统栈或信号栈上执行时，这会返回当前M的g0
或gsinal.

To determine if you're running on the user stack or the system stack,
use `getg() == getg().m.curg`.
使用`getg() == getg().m.curg`可以判断当前是否执行在系统或信号栈上

Error handling and reporting
============================

Errors that can reasonably be recovered from in user code should use
`panic` like usual. However, there are some situations where `panic`
will cause an immediate fatal error, such as when called on the system
stack or when called during `mallocgc`.

Most errors in the runtime are not recoverable. For these, use
`throw`, which dumps the traceback and immediately terminates the
process. In general, `throw` should be passed a string constant to
avoid allocating in perilous situations. By convention, additional
details are printed before `throw` using `print` or `println` and the
messages are prefixed with "runtime:".

For runtime error debugging, it's useful to run with
`GOTRACEBACK=system` or `GOTRACEBACK=crash`.

Synchronization
===============

The runtime has multiple synchronization mechanisms. They differ in
semantics and, in particular, in whether they interact with the
goroutine scheduler or the OS scheduler.
runtime有多种同步机制. 其语义和是否与goroutine调度器和OS调度器有所区别.

The simplest is `mutex`, which is manipulated using `lock` and
`unlock`. This should be used to protect shared structures for short
periods. Blocking on a `mutex` directly blocks the M, without
interacting with the Go scheduler. This means it is safe to use from
the lowest levels of the runtime, but also prevents any associated G
and P from being rescheduled. `rwmutex` is similar.
最简单的是`mutex`. 这用来在短时间内保护共享数据结构. 阻塞`mutex`不直接与Go调度器
交流, 直接阻塞M. 这意味着runtime最底层使用是安全的, 使用这个也避免和关联的G和P被
重新调度.

For one-shot notifications, use `note`, which provides `notesleep` and
`notewakeup`. Unlike traditional UNIX `sleep`/`wakeup`, `note`s are
race-free, so `notesleep` returns immediately if the `notewakeup` has
already happened. A `note` can be reset after use with `noteclear`,
which must not race with a sleep or wakeup. Like `mutex`, blocking on
a `note` blocks the M. However, there are different ways to sleep on a
`note`:`notesleep` also prevents rescheduling of any associated G and
P, while `notetsleepg` acts like a blocking system call that allows
the P to be reused to run another G. This is still less efficient than
blocking the G directly since it consumes an M.
对于一次性的提醒, 使用`note`. `note`也会阻塞M. 但其仍有两种情况, notesleep
阻止关联的G和P重调度. `notetsleepg`运行P不被阻塞，继续被重用来执行其他G.
这仍然不够高效，因为阻塞了一个M.

To interact directly with the goroutine scheduler, use `gopark` and
`goready`. `gopark` parks the current goroutine—putting it in the
"waiting" state and removing it from the scheduler's run queue—and
schedules another goroutine on the current M/P. `goready` puts a
parked goroutine back in the "runnable" state and adds it to the run
queue.
为了直接与goroutine调度器直接交流, 使用`gopark`和`goready`. `gopark`停止当前
goroutine, 将其放入等待状态，将其从调度器的运行队列中移除，然后调度另一个goroutine
在当前M/P上执行.

In summary,

<table>
<tr><th></th><th colspan="3">Blocks</th></tr>
<tr><th>Interface</th><th>G</th><th>M</th><th>P</th></tr>
<tr><td>(rw)mutex</td><td>Y</td><td>Y</td><td>Y</td></tr>
<tr><td>note</td><td>Y</td><td>Y</td><td>Y/N</td></tr>
<tr><td>park</td><td>Y</td><td>N</td><td>N</td></tr>
</table>

Unmanaged memory
================

In general, the runtime tries to use regular heap allocation. However,
in some cases the runtime must allocate objects outside of the garbage
collected heap, in *unmanaged memory*. This is necessary if the
objects are part of the memory manager itself or if they must be
allocated in situations where the caller may not have a P.

There are three mechanisms for allocating unmanaged memory:

* sysAlloc obtains memory directly from the OS. This comes in whole
  multiples of the system page size, but it can be freed with sysFree.

* persistentalloc combines multiple smaller allocations into a single
  sysAlloc to avoid fragmentation. However, there is no way to free
  persistentalloced objects (hence the name).

* fixalloc is a SLAB-style allocator that allocates objects of a fixed
  size. fixalloced objects can be freed, but this memory can only be
  reused by the same fixalloc pool, so it can only be reused for
  objects of the same type.

In general, types that are allocated using any of these should be
marked `//go:notinheap` (see below).

Objects that are allocated in unmanaged memory **must not** contain
heap pointers unless the following rules are also obeyed:

1. Any pointers from unmanaged memory to the heap must be added as
   explicit garbage collection roots in `runtime.markroot`.

2. If the memory is reused, the heap pointers must be zero-initialized
   before they become visible as GC roots. Otherwise, the GC may
   observe stale heap pointers. See "Zero-initialization versus
   zeroing".

Zero-initialization versus zeroing
==================================

There are two types of zeroing in the runtime, depending on whether
the memory is already initialized to a type-safe state.

If memory is not in a type-safe state, meaning it potentially contains
"garbage" because it was just allocated and it is being initialized
for first use, then it must be *zero-initialized* using
`memclrNoHeapPointers` or non-pointer writes. This does not perform
write barriers.

If memory is already in a type-safe state and is simply being set to
the zero value, this must be done using regular writes, `typedmemclr`,
or `memclrHasPointers`. This performs write barriers.

Runtime-only compiler directives
================================

In addition to the "//go:" directives documented in "go doc compile",
the compiler supports additional directives only in the runtime.

go:systemstack
--------------

`go:systemstack` indicates that a function must run on the system
stack. This is checked dynamically by a special function prologue.

go:nowritebarrier
-----------------

`go:nowritebarrier` directs the compiler to emit an error if the
following function contains any write barriers. (It *does not*
suppress the generation of write barriers; it is simply an assertion.)

Usually you want `go:nowritebarrierrec`. `go:nowritebarrier` is
primarily useful in situations where it's "nice" not to have write
barriers, but not required for correctness.

go:nowritebarrierrec and go:yeswritebarrierrec
----------------------------------------------

`go:nowritebarrierrec` directs the compiler to emit an error if the
following function or any function it calls recursively, up to a
`go:yeswritebarrierrec`, contains a write barrier.

Logically, the compiler floods the call graph starting from each
`go:nowritebarrierrec` function and produces an error if it encounters
a function containing a write barrier. This flood stops at
`go:yeswritebarrierrec` functions.

`go:nowritebarrierrec` is used in the implementation of the write
barrier to prevent infinite loops.

Both directives are used in the scheduler. The write barrier requires
an active P (`getg().m.p != nil`) and scheduler code often runs
without an active P. In this case, `go:nowritebarrierrec` is used on
functions that release the P or may run without a P and
`go:yeswritebarrierrec` is used when code re-acquires an active P.
Since these are function-level annotations, code that releases or
acquires a P may need to be split across two functions.

go:notinheap
------------

`go:notinheap` applies to type declarations. It indicates that a type
must never be allocated from the GC'd heap. Specifically, pointers to
this type must always fail the `runtime.inheap` check. The type may be
used for global variables, for stack variables, or for objects in
unmanaged memory (e.g., allocated with `sysAlloc`, `persistentalloc`,
`fixalloc`, or from a manually-managed span). Specifically:

1. `new(T)`, `make([]T)`, `append([]T, ...)` and implicit heap
   allocation of T are disallowed. (Though implicit allocations are
   disallowed in the runtime anyway.)

2. A pointer to a regular type (other than `unsafe.Pointer`) cannot be
   converted to a pointer to a `go:notinheap` type, even if they have
   the same underlying type.

3. Any type that contains a `go:notinheap` type is itself
   `go:notinheap`. Structs and arrays are `go:notinheap` if their
   elements are. Maps and channels of `go:notinheap` types are
   disallowed. To keep things explicit, any type declaration where the
   type is implicitly `go:notinheap` must be explicitly marked
   `go:notinheap` as well.

4. Write barriers on pointers to `go:notinheap` types can be omitted.

The last point is the real benefit of `go:notinheap`. The runtime uses
it for low-level internal structures to avoid memory barriers in the
scheduler and the memory allocator where they are illegal or simply
inefficient. This mechanism is reasonably safe and does not compromise
the readability of the runtime.
