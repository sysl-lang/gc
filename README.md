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

An object's payload is **zeroed by `alloc`**, on the free-list path as well as the bump path, and
that is what a client may rely on when it stores anything reference counted. (`alloc_raw`, below, is
the one entry point that does not, and everything in this section is what it hands to the caller.)

**A reference-counted value is fine, and its finaliser is how it is given back.** A `string`, a `Buf`
and a `Map` all work: store one in, release it by writing an empty one in `finalize`. That is what
lets a language's string be collected — the object is the sysl string's only owner, and finalising
takes its count to zero — without this package reimplementing string handling over raw blocks.

**The zeroing is what makes that true, and until 0.2.1 it was not done.** Assigning a `string`, a
`Buf` or a `Map` in sysl *releases the previous occupant*, so the client's first write into a payload
releases whatever bytes were already there — a dead object's on a reused block, the allocator's
leavings on a `malloc`ed heap. The release lands on something that is not a live reference and the
process takes a bus error later, inside a tracer or a finaliser, a long way from the assignment that
caused it. `sysl-lang/slate` hit exactly this: a crash in its own `mark_map` on roughly one run in
three, which survived at all only because its heap is a zeroed static and so only the free-list path
was ever dirty.

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

## `alloc_raw`, for a constructor that writes every byte anyway

`alloc` zeroes the payload and a client's constructor then overwrites it, which is the same bytes
written twice. `sysl-lang/slate`'s profile made that visible: `memset` plus `__bzero` was 1.87% of
its whole run, and `new_object` did not appear under its own name at all — the zeroing *is* what a
construction costs outside `alloc`. So, since 0.2.5:

```sysl
var o: *Obj = ptr_cast(gc.alloc_raw(h, sizeof(Obj), &obj_kind))
o.tag = tag
for i in 0..<4
    o.props[i] = null        // every byte, before the next allocation
```

Everything `alloc` does, `alloc_raw` does — the class, the split, the object list, the accounting,
`null` on a full heap — except clearing the bytes. **What the caller takes on in exchange is two
things, and the first is not the obvious one:**

- **Every byte of the payload is written before the next allocation and before the next collection.**
  The collector is a reader of this object that the caller does not schedule: the moment a collection
  starts, the object's `trace` runs over whatever is in the payload, and a half-written payload is a
  tracer following something that is not a pointer. "I will fill the rest in shortly" is not safe
  here in the way it would be with a single-threaded reader.
- **No reference-counted member may be assigned into it first**, for the reason the section above
  gives: the assignment releases the previous occupant, and the previous occupant is garbage. Such a
  member is *initialized* by writing its raw bytes, or the object comes from `alloc`.

**`alloc` is the right entry point unless a measurement says otherwise.** The win exists only where
the caller was going to write the whole payload regardless — an interpreter's object constructor,
and not much else. On this package's own allocate-and-drop benchmark, with the payload constructed
in both arms so that the comparison is honest, `alloc_raw` is **6.4%** faster than `alloc`; a
benchmark that allocates and never writes makes it look like 30%, and that number is not about
anything real.

## The four hooks

`Kind` carries four, of which only the first is required. Three `null`s is the ordinary case.

| hook | when it runs | what it is for |
|---|---|---|
| `trace` | during marking | grey what this object references |
| `finalize` | during the sweep that frees it | release what it owns outside the heap |
| `weaken` | after marking, before the sweep | clear slots whose targets did not survive |
| `ephemeron` | between marking passes | mark the values whose keys turned out live |

**`finalize` is handed the payload and not the heap, and that is the contract rather than an
oversight.** A finalizer runs while the sweep is walking the block, so it may not allocate, may
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

**Since 0.2.6 each of those two passes runs only where a live object's kind has its hook.** Both are a
walk of the whole object list, live and dead, and for a program holding no weak value they found
nothing on every object of every collection. The heap counts the live objects whose kind carries a
`weaken` and those carrying an `ephemeron` — raised as an object is handed out, lowered as the sweep
frees one, zeroed by `dispose` — and a pass whose count is zero is not run. At one or more it runs
exactly as it always did. `weak_objects`, `ephemeron_objects` and `weak_walks` read the two counts and
how many times a weak pass has walked the list, which is what the tests assert on.

**So a client that keeps a spare weak object alive for ever pays both passes for ever.** An
interpreter that reserves one of each kind for an out-of-memory path should take the reserve under a
kind without the hooks; measured in slate's `csv` benchmark, the collector went from 58 ms to 40 ms
once its two reserves stopped counting.

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
if gc.live_size(&h) > threshold
    gc.collect(&h, &roots)
    threshold = max(Floor, gc.live_size(&h) * 2)
```

The cost of waiting is that peak memory runs higher than it strictly needs to. The benefit is that
the entire class of "the collector ran while I was holding a raw pointer in a local" cannot happen.

### Threshold on `live_size`, and grow it by what survived

**Both halves of those two lines matter, and this README recommended the wrong number until 0.2.2.**

`live_size` is the bytes currently held; `used` is the high-water mark and *never falls*, because a
sweep returns storage to the free list rather than to the block. So `if gc.used(&h) > threshold`
asks **"has this heap ever grown past N"** — which is true forever once it is true. The client then
runs a full mark-sweep at every dispatch boundary for the rest of the program, reclaiming almost
nothing each time and reporting nothing wrong. slate found this the expensive way: a 20,000-iteration
loop ran **19,629 collections**, and the only symptom was that it was slow.

**A fixed threshold is the other half of the same mistake**, and it survives the first fix. Once a
program's genuine live set sits above the constant, every collection is immediately followed by
another. So raise the threshold to a multiple of what actually survived — the standard rule, and
what makes collection cost proportional to garbage produced rather than to statements executed.
A floor keeps it from thrashing on a small heap.

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

Mark and sweep. Stop the world. Non-moving. Free lists segregated by size class, splitting, and
neighbours merged during the sweep. No generations and no write barrier.

**Non-moving is what makes it easy to use, and it is not a consolation prize.** An object never
changes address, so a client may hold a raw `*T` to a collected object anywhere — in a local, in a
struct, across a call — with no handle table and without telling the collector. A copying collector
buys locality and pays for it by making every one of those illegal.

**The mark phase is a loop, not a recursion.** A worklist threaded through the object headers means a
list a million long and a tree a million deep both cost O(1) stack. That is not an optimization: a
recursive mark phase is a stack overflow waiting for a client's data to get large enough, and it is
the one place where being simple and being correct come apart. `sysl test .` marks a 20,000-deep
chain to hold the line.

## What an allocation costs

**A collector's sweep hands back tens of thousands of blocks at a stroke, so the free list is where a
tracing collector's allocator gets into trouble.** Searching one list for a block that fits makes an
allocation cost what the last collection freed — which is to say it gets slowest exactly when it has
just been given the most to work with. `sysl-lang/slate` measured it at **266 µs per allocation**
against a heap holding 115,000 swept blocks, where the same allocations on a fresh heap cost 8 ms in
total, and a profiler put 1,421 of 1,426 samples inside `alloc`.

Since **0.2.3** the free storage is segregated by size class instead:

- **Below a kilobyte, one bin per exact size**, sixteen bytes apart — the granularity every block is
  rounded to anyway. A bin holds blocks of one size, so its head either is the answer or the bin is
  empty: nothing is examined and rejected, and the common case is a single pointer read.
- **Above a kilobyte, one bin per power of two.** A bin like that spans a range, so at most eight of
  its blocks are examined; failing that, the head of any larger class serves, since every block there
  is bigger than anything the request's class could hold.
- **A block bigger than the request is split** and the remainder goes back on the list for its own
  class, so a freed 4 KB block does not vanish into a 32-byte object.
- **Neighbours that are free at the end of a sweep are merged into one block.** Nothing but an
  address-ordered pass can see that two blocks are adjacent, so the sweep IS that pass: it walks the
  block from its base to the bump pointer once, frees each unmarked object as it meets it, relinks
  each survivor onto the object list in the order met, and closes every run of adjacent free blocks
  into its bin.

- **Since 0.2.7 that is one walk, not two.** The sweep used to follow the object list, which is in
  allocation order and, once storage is being reused, scattered across the block — a dependent load
  from wherever the next object happened to be — and then walk the block a second time to rebuild the
  bins. Folding the first into the second leaves one sequential read of each header. Per dead object
  what is touched is its size, its mark, its kind's `finalize` (called only where it is not null),
  and the kind's two weak hooks only while a weak object is live at all. On 200,000 objects per
  collection with one in ten surviving, a collection went from about 30 ns per dead object to 3–7,
  with or without a finalizer; `sysl-lang/slate`'s collector time fell 20–25% on its
  allocation-dense benchmarks.
- **Since 0.2.4, "which is the next class above this one that has a block" is a bit scan**, not a
  walk up the array. The heap carries two 64-bit words with a bit per class, set exactly when the
  class has a block, and the search is a shift and a `trailing_zeros`.

- **Since 0.2.5, "which class is this block" is a bit scan too.** Finding which power of two a size
  falls in by shifting it right until it is one costs up to twenty-three shifts and twenty-three
  compares; `leading_zeros` costs one instruction. It is easy to believe only a large request pays
  for that loop, and that is the mistake: a heap in its steady state is one big free block, so an
  *ordinary small* allocation splits it and pushes the large remainder back, and the remainder is
  what the loop runs on. `sysl-lang/slate`'s sampled profile put that loop's `j += 1` as the single
  hottest instruction inside `alloc`. Worth **6.7%** of an allocate-and-drop benchmark, best of five.

  The width of a `usize` here comes from `sizeof`, not from `count_ones() + count_zeros()`. The
  second is the portable spelling `sysl.slices` uses and it is correct, but it is two population
  counts of a *runtime* value, and arm64 has no scalar popcount — written that way the rewrite came
  out **slower** than the loop it replaced, which is the version that was measured first.

  **That search is the steady state rather than the exception, which is why it was worth a word of
  storage.** The two bullets above see to it: a sweep coalesces neighbours into large blocks and a
  split hands remainders back to large classes, so a program allocating small objects finds its own
  class empty almost every time and starts climbing. Climbing 87 classes meant 87 dependent loads
  per allocation. Measured on an allocate-and-drop loop at an interpreter's shape — 20 million
  40-byte payloads over a heap collected between rounds — 0.2.4 is **2.35× faster than 0.2.3**, best
  of five. `sysl-lang/slate`'s profile had put 6.2% of its whole run time in that one walk.

The same benchmark — sweep 115,000 × 192-byte blocks, each separated from the next by a survivor so
that none of them can merge, then allocate 16,000 objects of a size that fits none of them — runs in
**under 60 ms where 0.2.2 took 21.9 s** — and that whole figure includes building the 230,000
objects and collecting over them, which the 21.9 s barely does.

`gc.alloc_steps(&h)` answers how many free blocks the allocator has examined since the heap was made.
It exists so that this is a property a test can assert on rather than a wall-clock measurement: the
suite fills a heap, sweeps 100,000 blocks out of it, and requires that 10,000 following allocations
examine a bounded number of blocks between them. A client can read the same number to find out
whether its own workload is paying for a search.

## The costs, named

- **The sweep walks the whole block, not just the objects.** It costs a step per block, free ones
  included, which is the price of never fragmenting.
- **A large class is a range, so it is searched.** At most eight of its blocks are examined before
  the request is served out of a larger class instead, and the rest of that class is walked only
  when the heap is otherwise full and the answer would be `null`.
- **Stop the world.** Collection pauses for as long as the live set takes to trace.
- **The ephemeron pass is O(entries × passes).** It re-walks every live object with the hook until a
  pass changes nothing. Weak maps are usually few and small; a program with many large ones would
  feel it. A heap holding no live object with the hook skips the pass entirely.
- **One block, fixed at startup.** There is no growing. `alloc` answers `null` when the block is
  full, and a caller that ignores that will write through it.

Each of those is a thing to fix if it ever matters, and none of them is load-bearing on the
interface.

## Tests

```
sysl test .
```

44 of them, and the ones that matter most are checked by falsification rather than by passing: with
the weak map's values marked strongly instead of through the ephemeron hook,
`ephemeron_dead_key_drops_entry` and `ephemeron_self_reference_is_collected` both fail, and
`alloc_does_not_walk_the_whole_free_list` was written against 0.2.2's allocator and failed there
before the size classes existed. A test that could not have failed is not evidence.

The five that came with the class mask are checked the same way. `assert_mask_agrees` reads both
views of every class — the bit and the bin — and is run after each allocation, each sweep and each
split, so a maintenance site left out of any of the five places that assign a bin fails it; the
scan's own edges (the lowest class, the highest, and the boundary between the two words, where a
shift by a whole word width would be undefined) are driven directly rather than through a heap,
because no request can reach a 16-byte class or an 8 GB one.

The seven that came with the weak-pass counts are checked the same way: with both passes forced off,
ten tests fail, every weak-reference and weak-map test among them.
