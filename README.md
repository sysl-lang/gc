# gc

A tracing garbage collector for sysl, over storage the caller supplies.

Reference counting cannot free a cycle, and `weak` only helps where the back-edge is known when it is
written. An interpreter running somebody else's program does not have that: a JavaScript closure
capturing the scope that holds it is a cycle nobody declared. So this traces.

```sysl
import sh.sysl.gc

// One block, from anywhere. A static array on a board, a malloc on a laptop.
var storage: [65536]u8 = [0u8; 65536]
var h = gc.heap(&storage[0], 65536, null)
```

The third argument is the heap's context: the client's own state, which a hook reads back with
`gc.context(h)`. `null` is fine for a client that keeps none.

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

**Payloads are 16-byte aligned whatever `base` is.** `heap` enters the block up to 15 bytes in, so
that the first payload lands on a 16-byte boundary, and every block after it is a multiple of the
16-byte grain. Up to 15 bytes of `cap` are given up to that. A block no bigger than the bytes given
up holds nothing, and every `alloc` on it answers `null`. A client may therefore keep a `u128`, a
SIMD vector or any 16-aligned type in a payload.

## What a collected object may hold

An object's payload is **zeroed by `alloc`**, on the free-list path as well as the bump path, and
that is what a client may rely on when it stores anything reference counted. (`alloc_raw`, below, is
the one entry point that does not, and everything in this section is what it hands to the caller.)

**A reference-counted value is fine, and its finaliser is how it is given back.** A `string`, a `Buf`
and a `Map` all work: store one in, and release it in `finalize`. That is what lets a language's
string be collected — the object is the sysl string's only owner, and finalising takes its count to
zero — without this package reimplementing string handling over raw blocks.

**Release a `Buf` or a `Map` with `.release()`, never by writing an empty one.** A `Buf` and a `Map`
are references to a core, so `x = buf()` or `x = map()` releases the old core and then *allocates* a
new, empty one into the payload. Nothing ever releases that one: the next `alloc` of the block zeroes
it. That is one leaked core per swept object. `x.release()` gives the core back and leaves nothing
behind. A `string` may be emptied with `x = ""`, because a literal is never allocated.

**The zeroing is what makes that true.** Assigning a `string`, a `Buf` or a `Map` in sysl *releases
the previous occupant*, so the client's first write into a payload releases whatever bytes were
already there — a dead object's on a reused block, the allocator's leavings on a `malloc`ed heap.
Without the zeroing the release lands on something that is not a live reference and the process
takes a bus error later, inside a tracer or a finaliser, a long way from the assignment that caused
it.

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
written twice: in an interpreter's profile `memset` plus `__bzero` was 1.87% of the whole run, and the
object constructor did not appear under its own name at all — the zeroing *is* what a construction
costs outside `alloc`. So:

```sysl
var o: *Obj = ptr_cast(gc.alloc_raw(h, sizeof(Obj), &obj_kind))
o.tag = tag
for i in 0..<4
    o.props[i] = null        // every byte, before the next allocation
```

Everything `alloc` does, `alloc_raw` does — the class, the split, the accounting,
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

Every hook is handed the payload and the heap: `trace`, `finalize` and `weaken` as `(*u8, *Heap) ->
unit`, `ephemeron` as `(*u8, *Heap) -> bool`.

**`finalize` is handed the heap so that it can reach `context(h)`**, which is how a runtime's
finalizer gives a handle back to the runtime without a global. That is all the heap is for there: a
finalizer runs while the sweep is walking the block, so it may not allocate, may not mark and may not
resurrect what it is finalizing.

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

**Each of those two passes runs only where a live object's kind has its hook.** Both walk the whole
heap in address order, live and dead, and for a program holding no weak value they would find nothing
on every object of every collection. The heap counts the live objects whose kind carries a `weaken`
and those carrying an `ephemeron` — raised as an object is handed out, lowered as the sweep frees
one, zeroed by `dispose` — and a pass whose count is zero is not run. `weak_objects`,
`ephemeron_objects` and `weak_walks` read the two counts and how many times a weak pass has walked
the heap, which is what the tests assert on.

**So a client that keeps a spare weak object alive for ever pays both passes for ever.** An
interpreter that reserves one of each kind for an out-of-memory path should take the reserve under a
kind without the hooks; in one interpreter benchmark the collector went from 58 ms to 40 ms once its
two reserves stopped counting.

**Finalizers and weak passes run in address order**, not allocation order.

`dispose` finalizes everything still live and empties the heap. Without it, a finalizer that closes a
file or releases a `Buf` never runs for anything still reachable when the program stops.

## The context

```sysl
heap(base: *u8, cap: usize, ctx: *u8) -> Heap
context(h: *Heap) -> *u8
```

`ctx` is whatever the client's hooks need to find its own state — an interpreter's `Vm`, a runtime's
shared tables — and `context` hands it back with one load. **It is per heap, not per thread**: one
heap has one context whatever thread is running, there is no global and no thread-local, and there is
no setter, so the context is fixed when the heap is made. The collector never reads through it.

A client that already finds its state another way passes `null`. One that embeds the `Heap` in its own
state struct can recover that struct from `h` by subtracting the field's `offsetof`, which costs no
load at all.

## An object's kind, and a client's own descriptors

```sysl
const KIND_BACK: usize = 2 * sizeof(usize)
kind_of(p: *u8) -> *Kind
```

`kind_of(p)` answers the `*Kind` the object was allocated with, read from `p - KIND_BACK` in one load.
**`KIND_BACK` is frozen for 0.3.x** — 16 bytes on a 64-bit target, 8 on a 32-bit one — so generated
code may load an object's kind inline without calling anything.

**The `Kind` layout is frozen too: four pointer-sized hooks, `trace`, `finalize`, `weaken`,
`ephemeron`, in that order.** That is what lets a client begin its own per-type descriptor with a
`Kind` and extend it past the `Kind` as far as it likes — a variant tag, a field count, a name, a
table of offsets. It hands `alloc` the address of the descriptor, and `kind_of` gives the descriptor
back:

```sysl
struct Descriptor
    kind: gc.Kind          // offset 0
    tag: usize
    refs: usize

@assert(offsetof(Descriptor, kind) == 0, "a descriptor begins with its Kind")

trace(p: *u8, h: *gc.Heap)
    val d: *Descriptor = ptr_cast(gc.kind_of(p))
    ...

var p = gc.alloc(h, size, &d.kind)
```

One tracer then serves every type, reading what to trace from the descriptor, and a `match` on a
variant reads its tag through the same load.

## Pointers outside the arena, and constants a client builds

**`mark` ignores any pointer that is not into the arena's used part**: `null`, an address below the
arena, an address past what has been handed out, another heap's object. It reads nothing for one —
not even the two words before it — so a tracer may hand `mark` every field it holds without asking
which are heap objects. `is_live` answers `false` for `null` and **`true` for any other pointer
outside the arena**, because the collector does not own it and it never dies; a weak slot holding one
is never cleared.

The test is one subtraction and one compare: the header's offset from the arena's base, computed with
wrap, against the bump pointer. An address below the base, `null` among them, wraps to far above it.
The payload of a zero-byte object at the very top of the used part is the used part's end, and that
is in range.

**That is what lets a client keep constant objects outside the heap**, in its data segment, and mix
them freely with heap objects. The contract:

- **`kind_of(p)` is valid on any `p` whose `p - KIND_BACK` word holds a `*Kind`**, inside the arena
  or out. A client constant is laid out with a 16-byte prefix (8 on a 32-bit target) ahead of its
  payload: the `*Kind` (or a descriptor beginning with one), then a word the client leaves `0`.
- **The kind word is the only header word a client may forge.** For a `p` outside the arena, gc
  never reads that word or the word after it, and never writes anything there.
- `mark` and `is_live` take such a `p` as above: ignored, and live.

```sysl
struct Constant
    kind: *gc.Kind
    word: usize            // the collector's; left 0
    value: Box

var one: Constant = Constant(&box_descriptor.kind, 0, Box(1))
val p: *u8 = ptr_cast(&one.value)    // gc.kind_of(p) == &box_descriptor.kind; gc.mark(h, p) is a no-op
```

A pointer *into* the arena that is not an object's payload — into the middle of one, or at a freed
block — is still not a legal argument; the `checks` option traps on it.

## Checks

```hocon
gc { git = "github.com/sysl-lang/gc", version = "0.3.0", options = { checks = true } }
```

The `checks` option (default `false`) validates the heap as it runs and traps on the first fault:

- `alloc` / `alloc_raw` with a `null` kind, or a kind whose `trace` is `null`;
- `mark` / `is_live` of a pointer that is not the payload of a live object of this heap — a freed
  block, a pointer into the middle of a payload, a misaligned one;
- after every sweep and every `dispose`: the blocks tile the used part of the arena exactly, no two
  free blocks are neighbours, every free block is on the bin for its size and every bin holds only
  its own class, and the class mask agrees with the bins.

With checks on, **`foreign_marks(h)`** also counts the non-null pointers outside the arena `mark` has
been handed since the heap was made. It is a statistic, never a trap: a client with no constants
outside the heap expects it to read zero, and a client with some can see how many marks they cost.
With checks off it is not kept and reads zero.

**Off, it costs nothing.** Every guarded site is `if option_checks then …` over a constant, so at
`false` the code is not compiled at all. The traps are `trap()`, which needs no capability, so a
checked build still runs on a bare-metal target.

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

**Both halves of those two lines matter.**

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

**Every object costs a three-word header**: 24 bytes on a 64-bit target, 12 on a 32-bit one — one
link (a free block's bin link, or a marked object's worklist link; never both at once), the `*Kind`,
and one word holding the block's length with the mark and free flags in its low bits. A block is the
payload plus the header, rounded up to 16 bytes, so an 8-byte payload takes 32 bytes, a 16- or
24-byte payload 48 and a 32-byte payload 64. There is no list of every object: the passes that visit
them all — the weak passes, the sweep, `dispose` — walk the arena in address order. The header is
private apart from `KIND_BACK`: the kind and the length word are the last two words before the
payload in every layout.

**The mark phase is a loop, not a recursion.** A worklist threaded through the object headers means a
list a million long and a tree a million deep both cost O(1) stack. That is not an optimization: a
recursive mark phase is a stack overflow waiting for a client's data to get large enough, and it is
the one place where being simple and being correct come apart. `sysl test .` marks a 20,000-deep
chain to hold the line.

## What an allocation costs

**A collector's sweep hands back tens of thousands of blocks at a stroke, so the free list is where a
tracing collector's allocator gets into trouble.** Searching one list for a block that fits makes an
allocation cost what the last collection freed — which is to say it gets slowest exactly when it has
just been given the most to work with. Against a heap holding 115,000 swept
blocks that costs **266 µs per allocation**, where the same allocations on a fresh heap cost 8 ms in
total.

So the free storage is segregated by size class:

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
  block from its base to the bump pointer once, frees each unmarked object as it meets it, and
  closes every run of adjacent free blocks into its bin.
- **The sweep is one sequential read of each header**, in address order, with no object list to
  chase. Per dead object what is touched is its size, its mark, its kind's `finalize` (called only
  where it is not null), and the kind's two weak hooks only while a weak object is live at all.
- **"Which is the next class above this one that has a block" is a bit scan**, not a walk up the
  array. The heap carries two 64-bit words with a bit per class, set exactly when the class has a
  block, and the search is a shift and a `trailing_zeros`.
- **"Which class is this block" is a bit scan too**, `leading_zeros` rather than a shift loop. A heap
  in its steady state is one big free block, so an *ordinary small* allocation splits it and pushes
  the large remainder back, and the remainder is what the class computation runs on. The width of a
  `usize` comes from `sizeof`, not from `count_ones() + count_zeros()`: the second is two population
  counts of a *runtime* value, and arm64 has no scalar popcount.

  **That search is the steady state rather than the exception, which is why it is worth a word of
  storage.** A sweep coalesces neighbours into large blocks and a split hands remainders back to
  large classes, so a program allocating small objects finds its own class empty almost every time
  and starts climbing. Climbing 87 classes would mean 87 dependent loads per allocation.

A benchmark that sweeps 115,000 × 192-byte blocks, each separated from the next by a survivor so
that none of them can merge, then allocates 16,000 objects of a size that fits none of them, runs in
under 60 ms, including building the 230,000 objects and collecting over them. A single free list
searched linearly takes 21.9 s on it.

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

## Migrating from 0.2.10

- Change `gc.heap(base, cap)` to `gc.heap(base, cap, null)`, or pass the client's state where its
  hooks want `gc.context(h)`.
- Change each finalizer `finalize_x(p: *u8)` to `finalize_x(p: *u8, h: *gc.Heap)`; one that has no
  use for the heap ignores it.
- Do not rely on finalizers running in allocation order; they run in address order.
- Drop a client-side range test guarding `mark`; `mark` ignores pointers outside the arena and
  `is_live` answers `true` for them (`false` for `null`).
- Size accounting by object: the header is 24 bytes on a 64-bit target (12 on a 32-bit one), so
  recompute any layout or capacity that assumed the old header.
- Optionally move a descriptor pointer kept in a payload's first word into a `Kind`-first descriptor
  and read it with `kind_of`, which saves the word in every object.

The block a consumer writes:

```hocon
dependencies {
  gc { git = "github.com/sysl-lang/gc", version = "0.3.0" }
}
```

## Threads

A heap is one piece of shared state: its context, its counters, and its bins. A
client that allocates from several threads serializes `alloc` with its own lock and stops the world
for `collect`, which runs on one thread. The context is per heap, not per thread. Every counter —
`live`, `live_size`, `weak_objects`, `ephemeron_objects`, `alloc_steps` — is updated by the
allocation or the sweep that changes it, so each is exact whenever no `alloc` or `collect` is
running.

## Tests

```
sysl test .
sysl test . --option checks=true
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .
```

The suite runs both ways. With `checks` off the tests that exist to see a check trap say `skip`, so a
plain run reports what it did not exercise rather than passing them unearned; with it on they run,
and every other test runs with the heap validated after every sweep.

The ones that matter most are checked by falsification rather than by passing: with
the weak map's values marked strongly instead of through the ephemeron hook,
`ephemeron_dead_key_drops_entry` and `ephemeron_self_reference_is_collected` both fail, and
`alloc_does_not_walk_the_whole_free_list` fails against a single linear free list. A test that could not have failed is not evidence.

The five class-mask tests are checked the same way. `assert_mask_agrees` reads both
views of every class — the bit and the bin — and is run after each allocation, each sweep and each
split, so a maintenance site left out of any of the five places that assign a bin fails it; the
scan's own edges (the lowest class, the highest, and the boundary between the two words, where a
shift by a whole word width would be undefined) are driven directly rather than through a heap,
because no request can reach a 16-byte class or an 8 GB one.

The seven weak-pass-count tests are checked the same way: with both passes forced off,
ten tests fail, every weak-reference and weak-map test among them.

So are the header and the range test. With `mark` pushing an object whatever its mark bit says, the
traced-once tests fail (a self-cycle, a node with two parents, an ephemeron's re-mark); with the range
test reduced to a `null` check, the constant, the weak-reference-to-a-constant, the arena-edge and the
`foreign_marks` tests fail; and with the range measured from the payload rather than the header, the
arena's base is taken for an object. Each `checks` trap
test fails with the check it exercises removed, except the `null`-kind one: an allocation naming no
kind stops on the null load without the check too, and the check only moves the stop to before any
heap state has changed.
