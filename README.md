# gc

A tracing garbage collector for sysl, over storage the caller supplies.

Reference counting cannot free a cycle, and `weak` only helps where the back-edge is known when it is
written. An interpreter running somebody else's program does not have that: a JavaScript closure
capturing the scope that holds it is a cycle nobody declared. So this traces.

```sysl
import sh.sysl.gc

// One block, from anywhere. A static array on a board, a malloc on a laptop.
var storage: [65536]u8 = [0u8; 65536]
var h = gc.heap(&storage[0], 65536)
```

## What a client writes

Three things, and nothing else: a tracer per object kind, a function that greys the roots, and calls
to `alloc`.

```sysl
struct Obj
    tag: int
    props: [4]*u8

trace_obj(p: *u8, h: *gc.Heap)
    val o: *Obj = ptr_cast(p)
    for i in 0..<4
        gc.mark(h, o.props[i])

new_obj(h: *gc.Heap, tag: int) -> *Obj
    var o: *Obj = ptr_cast(gc.alloc(h, sizeof(Obj), &trace_obj))
    o.tag = tag
    for i in 0..<4
        o.props[i] = null
    o
```

The collector never learns what `Obj` is. Every object carries a pointer to the function that greys
what it references, which is the same type erasure sysl's own ARC uses to put a destructor behind a
hook: one code path, no static type, a heap that may hold anything.

## Why it runs anywhere sysl runs

**The hard part of a portable collector is finding the roots, not marking them.** A conservative
collector — Boehm's is the one everybody reaches for — scans the machine stack, the registers and the
static data segment for anything shaped like a pointer. Every one of those is a per-target question
answered by a linker script and a register spill, which is why such a collector is a porting project
rather than a dependency.

This one never scans anything. The client hands over a function that greys its roots, because an
interpreter already knows where they are — its value stack, its scope chain, its globals. What is
left needs no operating system, no libc, no threads and no memory mapping: only a block of bytes.
That is why the manifest can state `requires {}` and mean it, and why there is no C in this package
to be missing a header on a freestanding target.

## Collect where you know what is live

**`alloc` never collects, and that is deliberate.** sysl has no stack maps, so an object reached only
from a sysl local is invisible to a mark phase that traces the client's roots. A collection triggered
from inside an allocation would sweep away the object its caller is halfway through building.

So a client collects at a point it chooses, where it knows what is live — the top of a statement
loop, or a bytecode dispatch boundary:

```sysl
roots(h: *gc.Heap)
    gc.mark(h, globals)
    for v in vm_stack
        gc.mark(h, v)

// ... at the dispatch boundary, not inside alloc:
if gc.used(&h) > threshold
    gc.collect(&h, &roots)
```

The cost of waiting is that peak memory runs higher than it strictly needs to. The benefit is that
the entire class of "the collector ran while I was holding a raw pointer in a local" cannot happen.

## Tracers live in a module, not in the entry file

**A tracer and a root function are reached by address, so both must be top-level functions.** In an
entry file — the one with top-level statements — anything declared after the first statement is
nested inside the program's body, and a nested function has no address to take:

```
error: 'roots' is a nested function, so it has no address to take — what would have to travel
beside the address is the frame it reads, and a '*extern' is one word. A top-level function is
what has an address
```

The interpreter's state has to be reachable from the root function, and in an entry file that state
would be a local of the body — so the two requirements collide there and nowhere else. Put the
object kinds, the tracers, the roots and the state in a **module**, where a top-level `var` is module
storage and every function has an address, and let the entry file just drive it:

```sysl
module interp

import sh.sysl.gc

var globals: *u8 = null

roots(h: *gc.Heap)
    gc.mark(h, globals)
```

This is the shape an interpreter wants anyway. It is called out because the collision is only visible
at the point you take the address, which is a long way from the `var` that caused it.

## What it is, exactly

Mark and sweep. Stop the world. Non-moving. One free list, first fit, no splitting and no coalescing.
No generations, no write barrier, no finalizers, no weak references.

**Non-moving is what makes it easy to use, and it is not a consolation prize.** An object never
changes address, so a client may hold a raw `*T` to a collected object anywhere — in a local, in a
struct, across a call — with no handle table and without telling the collector. A copying collector
buys locality and pays for it by making every one of those illegal.

**The mark phase is a loop, not a recursion.** A worklist threaded through the object headers means a
list a million long and a tree a million deep both cost O(1) stack. That is not an optimization: a
recursive mark phase is a stack overflow waiting for a client's data to get large enough, and it is
the one place where being simple and being correct come apart. `sysl test .` marks a 20,000-deep
chain to hold the line.

## The costs, named

- **First fit with no splitting.** A freed 4 KB block will serve a 32-byte request and the rest is
  wasted until that object dies. Fine for an interpreter whose objects cluster around a few sizes;
  not fine for wildly mixed ones.
- **No coalescing.** Two adjacent free blocks stay two blocks, so a long run of mixed sizes
  fragments and never un-fragments.
- **Stop the world.** Collection pauses for as long as the live set takes to trace.
- **One block, fixed at startup.** There is no growing. `alloc` answers `null` when the block is
  full, and a caller that ignores that will write through it.

Each of those is a thing to fix if it ever matters, and none of them is load-bearing on the
interface.

## Tests

```
sysl test .
```
