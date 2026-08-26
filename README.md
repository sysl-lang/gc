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

A `Kind` per object kind, a function that greys the roots, and calls to `alloc`.

```sysl
struct Obj
    tag: int
    props: [4]*u8

trace_obj(p: *u8, h: *gc.Heap)
    val o: *Obj = ptr_cast(p)
    for i in 0..<4
        gc.mark(h, o.props[i])

var obj_kind: gc.Kind = gc.Kind(&trace_obj, null, null, null)

new_obj(h: *gc.Heap, tag: int) -> *Obj
    var o: *Obj = ptr_cast(gc.alloc(h, sizeof(Obj), &obj_kind))
    o.tag = tag
    for i in 0..<4
        o.props[i] = null
    o
```

The collector never learns what `Obj` is. Every object points at the `Kind` its type shares, which is
the same type erasure sysl's own ARC uses to put a destructor behind a hook: one code path, no static
type, a heap that may hold anything.

## What a collected object may hold

An object's payload is **raw bytes**. `alloc` does not zero it, and on the free list it is a dead
object's bytes, so what a client stores there has to survive being written over memory the compiler
never initialised.

**A reference-counted value is fine, and its finaliser is how it is given back.** A `string`, a `Buf`
and a `Map` all work: store one in, release it by writing an empty one in `finalize`. That is what
lets a language's string be collected — the object is the sysl string's only owner, and finalising
takes its count to zero — without this package reimplementing string handling over raw blocks.

**A counted box `&T` is NOT fine, and the failure is a segfault rather than a diagnostic.**

```sysl
struct Holder
    k: &Payload

var o: *Holder = ptr_cast(gc.alloc(&h, sizeof(Holder), &kind))

o.k = boxed        // segfault
```

Zeroing the payload first does not help. A `&T` is non-nullable in sysl's model, so the release its
assignment performs on the previous occupant has nothing to guard uninitialised bytes with — and a
struct holding one is refused by the same rule, which is the case that bites, since a syntax tree
node is usually exactly that shape.

**The way round it is an index rather than a pointer.** Keep the boxed data in an ordinary
reference-counted structure outside the heap and store a `usize` into the object. Where the data is a
syntax tree that is the right arrangement anyway: a tree is immutable and acyclic, which is what
refcounting is good at, and one tree is shared by every closure made from it rather than copied into
each. `sysl-lang/slate` does this for its function bodies.

## The four hooks

`Kind` carries four, of which only the first is required. Three `null`s is the ordinary case.

| hook | when it runs | what it is for |
|---|---|---|
| `trace` | during marking | grey what this object references |
| `finalize` | during the sweep that frees it | release what it owns outside the heap |
| `weaken` | after marking, before the sweep | clear slots whose targets did not survive |
| `ephemeron` | between marking passes | mark the values whose keys turned out live |

**`finalize` is handed the payload and not the heap, and that is the contract rather than an
oversight.** A finalizer runs while the sweep is walking the object list, so it may not allocate, may
not mark and may not resurrect. Taking no `*Heap` puts all three out of reach, so the rule is enforced
by the signature instead of by a warning in a comment.

**`weaken` is what a `WeakRef` is.** An object whose `trace` does not mark its target, plus a `weaken`
that nulls the target once marking has decided the target is dead. It runs before anything is freed,
so a weak slot is never read after its target's storage has gone.

**`ephemeron` is what stops a weak map leaking, and its absence is silent.** A weak map holding
`key -> value` where the value refers back to its key is the ordinary shape — `wm.set(node, {owner:
node})` — and a collector that marks values strongly keeps that pair alive for as long as the map
lives, which is the exact leak a weak map exists to prevent. Marking them weakly is worse: a value
nothing else holds would be freed while its key is still in use. The answer is neither — a value is
marked *because* its key was marked, so marking must run again afterwards, since that value may be
some other entry's key. `collect` loops the hook and the worklist until a pass changes nothing.

`dispose` finalizes everything still live and empties the heap. Without it, a finalizer that closes a
file or releases a `Buf` never runs for anything still reachable when the program stops.

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

**This is a constraint on the client as well as a property of the collector.** Anything holding live
objects must be reachable from the root function — a coroutine suspended mid-`await`, a microtask
queue, a pending promise's reactions. In particular, **coroutines have to be heap-allocated frames
rather than native stacks**: a native stack per task is exactly the thing this collector will not
scan. That is the ordinary way to compile `async`/`await` anyway.

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

**Hooks are reached by address, so every one must be a top-level function.** In an entry file — the
one with top-level statements — anything declared after the first statement is nested inside the
program's body, and a nested function has no address to take:

```
error: 'roots' is a nested function, so it has no address to take — what would have to travel
beside the address is the frame it reads, and a '*extern' is one word. A top-level function is
what has an address
```

The interpreter's state has to be reachable from the root function, and in an entry file that state
would be a local of the body — so the two requirements collide there and nowhere else. Put the object
kinds, the hooks, the roots and the state in a **module**, where a top-level `var` is module storage
and every function has an address, and let the entry file just drive it. Module storage states its
type, so a `Kind` is written `var obj_kind: gc.Kind = ...`.

## What it is, exactly

Mark and sweep. Stop the world. Non-moving. One free list, first fit, no splitting and no coalescing.
No generations and no write barrier.

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
- **The ephemeron pass is O(entries × passes).** It re-walks every live object with the hook until a
  pass changes nothing. Weak maps are usually few and small; a program with many large ones would
  feel it.
- **One block, fixed at startup.** There is no growing. `alloc` answers `null` when the block is
  full, and a caller that ignores that will write through it.

Each of those is a thing to fix if it ever matters, and none of them is load-bearing on the
interface.

## Tests

```
sysl test .
```

16 of them, and the two that matter most are checked by falsification rather than by passing: with
the weak map's values marked strongly instead of through the ephemeron hook,
`ephemeron_dead_key_drops_entry` and `ephemeron_self_reference_is_collected` both fail. A test that
could not have failed is not evidence.
