# Node.js Thread Context: Discovering Context Records through the Async Context Frame

Define a Node.js-specific discovery mechanism for the **Thread-Local Context
Record** introduced by [OTEP 4947: Thread Context](4947-thread-ctx.md), so that
external readers can observe OpenTelemetry context in Node.js processes without
the SDK writing a thread-local on every context switch.

This proposal is the document promised by the ["Alternative for Node.js
support"](4947-thread-ctx.md#alternative-for-nodejs-support) section of OTEP
4947. It explains how the OTEP-4947 record format and the
[OTEP-4719](4719-process-ctx.md) process-context conventions can be discovered
and used within a Node.js application. Both are reused verbatim; only the
discovery mechanism is Node.js-specific.

## Motivation

[OTEP 4947](4947-thread-ctx.md) lists Node.js among the runtimes it does not
expect to support, on account of the high cost of FFI and the heavily
asynchronous nature of the runtime.

The consequence is that Node.js is the only runtime that OTEP surveys with no
out-of-process context mechanism at all: six of them get one from the
thread-local it specifies, and Go already has pprof labels, which readers
consume today under its own `go_pprof_labels_v1` schema version.

The reason is Node.js' concurrency model. Elsewhere the OS thread is a usable
handle, because a request occupies one thread for its duration. Node.js
interleaves many logical contexts on one thread per isolate, so "the context on
this thread" is not a useful concept there. It needs a discovery mechanism of
its own rather than an implementation of the existing one.

## How Node.js tracks the active continuation

What we really need to track, then, is the active continuation on a given
isolate. Fortunately Node.js already has a way to attach data to one, and the
mechanism acts on two levels:

* **V8** provides `ContinuationPreservedEmbedderData` (CPED), a per-isolate slot
  holding one value that V8 exchanges as it moves between continuations. V8
  attaches no meaning to the value; it only guarantees that the slot tracks the
  active continuation.
* **Node.js** decides what to put there. When its `AsyncLocalStorage` is backed
  by **`AsyncContextFrame`** (a Node.js construct, not a V8 one) the value in
  the CPED slot is an async-context frame, realized as a JavaScript `Map` from
  each live `AsyncLocalStorage` instance to its current store. Node.js also
  takes care to install the right `AsyncContextFrame` into the CPED slot on the
  continuation changes it effects itself (e.g. entering IO and time callbacks).

Said otherwise, CPED is the V8 mechanism and the async-context frame is Node's
application of it. An SDK, or any other Node.js tracing code, that puts its
record holder into an `AsyncLocalStorage` therefore gets context-switch tracking
for free, at exactly the granularity the runtime uses, with no native call on
attach or detach and no cost at all when nothing is attached.

What is left is to tell a reader how to walk from the isolate to the record.
That is what this proposal specifies.

## Explanation

"The SDK" below is shorthand for whichever component publishes the context — an
OpenTelemetry SDK, a vendor tracer, or any other Node.js tracing code. Such a
component publishes context by:

1. Creating one `AsyncLocalStorage` instance per isolate, and telling its native
   addon about it.
2. Allocating a **Thread-Local Context Record** (OTEP 4947's format, unchanged)
   behind a JavaScript wrapper object for every tracing span, and storing a raw
   pointer to the record in the wrapper's internal field.
3. Attaching context by storing that wrapper in the `AsyncLocalStorage` (and
   detaching it by storing `undefined`). Both are pure-JavaScript operations; no
   native code runs for them.

A reader walks:

```text
otel_thread_ctx_nodejs_v1 (TLS)  →  *cped_slot  →  AsyncContextFrame (a JS Map)
  →  look up the published AsyncLocalStorage instance as key
  →  the wrapper JSObject that is its value
  →  internal field 0  →  Thread-Local Context Record
```

A single ELF thread-local named `otel_thread_ctx_nodejs_v1` is still involved,
but it holds **discovery data, not a record pointer**. Its contents are fixed
for the life of the isolate, so it is written once at initialization rather than
on every context switch. This is the essential difference from OTEP 4947 and the
reason the mechanism is affordable in Node.js.

### Goals

* **Reuse the record format.** The bytes readers parse are identical to OTEP
  4947's, so readers share all record-parsing logic across runtimes. Only the
  code that arrives at a record differs.
* **No native call on the context-switch path.** Attach and detach must be
  ordinary `AsyncLocalStorage` operations.
* **Zero marginal cost when unused.** A process that never attaches context pays
  nothing beyond module load.
* **Reader flexibility.** As in OTEP 4947, any reader able to read
  `/proc/<pid>/maps` and target process memory should be able to implement this;
  nothing here is eBPF-specific.
* **Per-worker-thread correctness.** Each Node.js worker thread has its own
  isolate and must be independently observable.

### Non-goals

* Supporting Node.js versions or configurations where `AsyncLocalStorage` is not
  backed by `AsyncContextFrame` (see "Runtime requirements").
* Non-Linux platforms. As in OTEPs 4719 and 4947, the discovery contract is
  ELF/TLSDESC-based and Linux-specific.
* Attributing work that runs outside the isolate's own thread. Node.js
  dispatches filesystem, DNS, zlib and asynchronous crypto work to the libuv
  thread pool, and those threads run no JavaScript and host no isolate, so they
  never publish a discovery struct. Their struct stays zeroed, a reader finds
  the gate closed and reports no context. Such a thread would in fact suit OTEP
  4947's original mechanism well: it runs one work item at a time, so the thread
  genuinely is the unit of context for that item's duration. Unfortunately we
  have no way to submit a record to these threads. The record would have to be
  captured where the work is submitted, installed on the pool thread when the
  work item starts, cleared when it ends, and kept alive throughout even if the
  span owning it finished in the meantime. A submission mechanism would need to
  be built inside Node.js, so supporting this would mean changing Node.js
  itself, and is outside this proposal's scope.

## Internal details

### Process Context: Node.js Thread-Local Reference Data

As in OTEP 4947, process-scoped data is published as entries in
`ProcessContext.attributes` per OTEP 4719, so a reader reads it once rather than
per sample. This proposal uses the existing values with an alternate
`threadlocal.schema_version` of `nodejs_v1_dev`, and otherwise adds four more
process attributes.

Reused from OTEP 4947:

* `threadlocal.schema_version` — `nodejs_v1_dev` for experimentation, to become
  `nodejs_v1` once this OTEP is merged. A reader that recognizes this value
  knows to use the walk described here instead of OTEP 4947's TLS-pointer walk.
* `threadlocal.attribute_key_map` — unchanged, including its append-only
  semantics.

The four added attributes are all V8 layout constants captured from the V8
headers the addon was compiled against, so that a reader does not have to derive
them from the target's pointer-compression and sandbox build flags, nor look up
V8 internal symbols:

| Key | Meaning |
| :-- | :------ |
| `threadlocal.js_object_record_offset` | Byte offset, within the wrapper JSObject, of the slot holding the pointer to its record. That slot is internal field 0: JavaScript objects can be allocated with space for internal fields, which are typically used to hold pointers to native data structures. |
| `threadlocal.tagged_size` | V8's tagged-pointer width in bytes: 4 with pointer compression, 8 without. |
| `threadlocal.js_map_table_offset` | Byte offset, within a V8 `JSMap`, of the tagged pointer to its backing `OrderedHashMap` table. |
| `threadlocal.ordered_hash_map_header_size` | Size of the `OrderedHashMap` header preceding its element-count fields. |

Example:

```yaml
key: "threadlocal.schema_version"
value:
  string_value: "nodejs_v1_dev"

key: "threadlocal.attribute_key_map"
value:
  array_value:
    values:
      - string_value: "http.request.method"  # index 0
      - string_value: "http.route"           # index 1

key: "threadlocal.js_object_record_offset"
value:
  int_value: 24

key: "threadlocal.tagged_size"
value:
  int_value: 8

key: "threadlocal.js_map_table_offset"
value:
  int_value: 24

key: "threadlocal.ordered_hash_map_header_size"
value:
  int_value: 16
```

> **Note:** As in OTEP 4947, the `threadlocal.*` keys are inter-process
> coordination metadata rather than telemetry attributes, and are not expected
> to appear in OTLP exports.

The last three values are properties of the V8 build and not of this SDK; they
are published rather than hardcoded by readers because they vary with build
configuration, and because two of them (`js_map_table_offset`,
`ordered_hash_map_header_size`) are not exposed by V8's public headers and so
cannot be discovered by a reader at all without either this contract or its own
symbol archaeology.

Note that nothing published here describes the writer's own data structures.
Internal field 0 points straight at the record, and no writer-specific data
structures are part of the contract.

### Thread-Local Variable

A single thread-local, `otel_thread_ctx_nodejs_v1`, is exported as an ELF TLS
symbol in the dynamic symbol table, providing information specific to the
Node.js runtime to find the OTEP-4947 record. It is a struct, not a pointer:

| Name | Offset | Data type | Notes |
| :--- | :----- | :-------- | :---- |
| `cped_slot` | `0` | pointer | Address of this thread's isolate's `ContinuationPreservedEmbedderData` slot. The slot holds a tagged V8 word; dereferencing it yields the active Node.js `AsyncContextFrame`. Lets the reader reach the active frame without any V8 internal symbol lookup. Doubles as the **gate**: an all-zero value means the SDK has not published on this thread, has torn it down again, or has closed the gate temporarily, and no other field may be used while it reads zero. |
| `als_handle` | `sizeof(void *)` | pointer | A `v8::Global<Object>` referring to the published `AsyncLocalStorage` instance in this thread's isolate. Its representation is a single V8 internal pointer; dereference it to obtain the instance's tagged address, which is the key to look up in the frame. |
| `als_identity_hash` | `2 * sizeof(void *)` | int32, followed by 4 bytes of padding | The JS identity hash of that instance, so a reader can restrict its search to one hash bucket rather than scanning every entry. |
| `undefined_addr` | `3 * sizeof(void *)` | tagged word | This thread's isolate's tagged address of the `undefined` singleton. Lets the reader detect "no context attached" by comparison, rather than by structurally validating whatever the frame maps our key to. |

All four fields are fixed while the isolate lives, but they are not written only
once: the SDK populates them when it installs its hook and zeroes them again at
teardown. A reader MUST therefore re-read the struct each time it samples the
thread, and MUST NOT substitute cached field values — one holding a pre-teardown
copy would go on walking a dead isolate's `cped_slot`. That costs one read of
four words, which is negligible beside the walk it precedes.

Upon initialization implementations MUST write the nonzero `cped_slot` value
last, and upon isolate teardown they must write the zero `cped_slot` value
first, using compiler fences (`atomic_signal_fence` or equivalent) and volatile
writes to prevent instruction reordering by the compiler. This way a reader
gating on `cped_slot` can be guaranteed to always see a fully populated struct
when `cped_slot` is nonzero.

The TLS access-model requirements of OTEP 4947's "Thread-Local Variable
Resolution" apply unchanged: writers SHOULD use the TLSDESC dialect, and readers
MUST support Global Dynamic/TLSDESC, Global Dynamic/legacy GNU, and
linker-relaxed initial-exec or local-exec access.

Because Node.js pins each isolate to a thread and creates a fresh isolate per
worker thread, a thread-local struct is the natural home for this data: each
worker thread that installs the hook publishes its own `cped_slot`, its own
`AsyncLocalStorage` instance, and its own `undefined_addr`, and is independently
observable. Threads that never install the hook leave the struct zeroed, which
is what lets `cped_slot` serve as the gate: no live isolate has its CPED slot at
address zero.

A closed gate is not necessarily permanent. A writer MAY close it transiently,
for as long as some condition makes the walk unsafe — see "Garbage collection"
for the motivating example. Readers MUST therefore re-test the gate on every
sample, and MUST NOT infer from a zero reading that a thread is permanently
uninstrumented.

Because the contract is byte-level, "cleared" means an all-zero representation.
A C++ writer assigning a null pointer produces that on the ELF platforms in
scope, the same assumption OTEP 4947 already relies on; a writer on any platform
where a null pointer is not all-zero bits MUST zero the bytes explicitly.

### Thread-Local Context Record

Unchanged from OTEP 4947, including field offsets, `attrs-data` encoding, the
`valid` byte, the 2-byte alignment requirement, the last-occurrence-wins rule
for repeated key indexes, and the recommendation to keep the total record at or
under 640 bytes. Readers MUST be able to use the same parser for both.

### Publication Protocol

#### 1. Isolate initialization

On first use, per isolate, the SDK:

1. Verifies that `AsyncLocalStorage` is backed by `AsyncContextFrame` (see
   "Runtime requirements").
2. Creates one `AsyncLocalStorage` instance dedicated to this mechanism.
3. Populates `otel_thread_ctx_nodejs_v1` with the four fields above, writing
   `cped_slot` **last**, as a volatile store preceded by a compiler fence.
   Publishing the gate after everything it guards means a reader either sees
   zero and stops, or sees a fully populated struct.
4. Publishes the **Node.js Thread-Local Reference Data** into the Process
   Context per OTEP 4719. The publication to process context is idempotent, so
   it can be repeated on every isolate initialization, but an implementation MAY
   optimize for only publishing the process context data once per process.

The `AsyncLocalStorage` instance SHOULD NOT be exposed to application code, so
that nothing but the SDK can put values into the slot a reader trusts.

#### 2. Context attachment

When context becomes active, the SDK:

1. Allocates a **Thread-Local Context Record**, writes the trace context and any
   configured attributes into it, and sets `valid` to 1 last, ordered with a
   compiler fence and a volatile store as OTEP 4947 requires.
2. Allocates a new JavaScript object with an internal field, and stores a raw
   pointer to the record in that internal field. This publication step is what
   makes the record reachable; until it happens, no reader can observe a
   partially built record.
3. Stores the wrapper in the `AsyncLocalStorage`, for a scope or until replaced.

Step 3 is the only step on the hot path when a wrapper is reused, which is the
intended pattern: an SDK SHOULD cache the record and its wrapper created in
steps 1 and 2 on the span or equivalent object so that re-entering a context
allocates nothing and calls no native code.

The record's lifetime SHOULD be tied to the wrapper's, so that a record stays
alive exactly as long as the wrapper is reachable in JavaScript heap and can
thus still be presented to a reader. The record should be released when the
wrapper is known to be unreachable. (This is typically achieved using V8 Globals
as weak references to wrappers with garbage collection callbacks.)

#### 3. Context detachment

A wrapper stored in an `AsyncLocalStorage` stays reachable from every
async-context frame derived from the one it was stored in, so several frames can
present the same record at once. That gives detachment two mechanisms, for two
different scopes:

* **Detach from the current frame** — store `undefined` in the
  `AsyncLocalStorage`. This affects the current frame and frames derived from
  it. Sibling frames and detached continuations that hold the same wrapper
  reference are unaffected and continue to present the record. This is used when
  the span is not finished yet but is no longer associated with the current
  asynchronous execution.
* **Invalidate the record** — set the record's `valid` byte to 0 in place,
  ordered with a compiler fence and a volatile store. Because every frame
  holding the wrapper reference sees the same record, this single write drops it
  out of scope for all of them at once. This is used when the span ends.

An SDK SHOULD invalidate on span end rather than relying on detachment alone, or
an out-of-process reader may keep observing an ended span's identity on frames
that were never explicitly cleared. Regardless, an SDK MAY also detach from the
current frame when the span ends in addition to invalidating the record.

#### 4. Growing the attribute payload

OTEP 4947 permits appending to `attrs-data` in place, publishing the new extent
by writing `attrs-data-size` last. That applies here unchanged when the existing
allocation has room.

When it does not, a writer that has to move the record MUST re-publish the new
record's address into internal field 0 of the **same** wrapper object, ordered
after all writes to the new record, and MUST NOT replace the wrapper. Because
every frame reaches the record through the wrapper, this keeps the append
visible to all of them, with the internal-field store as the single atomic
boundary the reader observes. The old record may be released immediately
afterwards under OTEP 4947's signal-handler semantics, provided the release
cannot be reordered before that store.

#### 5. Isolate teardown

Before an isolate is torn down, the SDK MUST clear the thread-local, and MUST
clear `cped_slot` **first**, as a volatile store followed by a compiler fence.
It SHOULD additionally clear internal field 0 of all live wrappers known to it.

Clearing wrapper internal fields is proportional to the number of live wrappers
and requires the SDK to track them all; it is defense in depth, and is redundant
once the thread-local is cleared, since a reader that stops at `cped_slot == 0`
never reaches a wrapper.

Neither omission can crash a reader as reading freed or unmapped memory in
another process fails or returns garbage rather than faulting the reader. The
risk is misattribution instead — see "Reader-visible consequences of incomplete
teardown" below.

### Reading Protocol

As in OTEP 4947, the reader MUST only read while the target thread is stopped or
interrupted.

#### 1. Process initialization

Unchanged from OTEP 4947's process-initialization steps, except that the reader
looks for `otel_thread_ctx_nodejs_v1` in the dynamic symbol tables, and treats a
`threadlocal.schema_version` of `nodejs_v1` as selecting this walk. A reader
MUST also read the four V8 layout constants before sampling; they are not
optional for this schema, and a reader that finds `schema_version` set to
`nodejs_v1` without them SHOULD treat the process context as incomplete and
re-read it on the next update.

#### 2. Thread sampling

The pseudo-code below assumes a 64-bit build with pointer compression and the V8
sandbox both off, which is true of Node's bundled V8 in the versions this
mechanism supports. Tagged pointers have their low bit set; clear it to get the
object address.

```cpp
auto* ctx = read_tls<otel_thread_ctx_nodejs_v1_t>();
if (ctx->cped_slot == 0) return NO_CONTEXT;  // nothing published here
// No async-context frame is active.
if (*ctx->cped_slot == ctx->undefined_addr) return NO_CONTEXT;

// CPED -> active AsyncContextFrame (a JS Map) -> its backing OrderedHashMap.
auto* acf = untag<JSMap>(*ctx->cped_slot);
auto* table = untag<OrderedHashMap>(
    *(tagged_ptr*)((char*)acf + js_map_table_offset));

// Find the entry keyed by our AsyncLocalStorage instance. A reader may use the
// published identity hash to walk a single bucket; a simpler one may scan all
// entries. Bucket and entry layout follow from ordered_hash_map_header_size and
// tagged_size.
uintptr_t als = *ctx->als_handle;
Entry* e = find_entry(table, als, ctx->als_identity_hash);
if (!e) return NO_CONTEXT;  // not in this frame
if (e->value == ctx->undefined_addr) return NO_CONTEXT;  // explicitly detached

// The value is the wrapper JSObject; internal field 0 holds the record pointer.
auto* wrapper = untag<JSObject>(e->value);
auto* record =
    *(OtelThreadCtxRecord**)((char*)wrapper + js_object_record_offset);
if (record == nullptr) return NO_CONTEXT;  // teardown in progress
if (record->valid != 1) return NO_CONTEXT;  // invalidated or mid-update
// Parse exactly as in OTEP 4947 from here on.
```

Both `NO_CONTEXT` exits before the `JSMap` walk are cheap, and the second is the
common case for a process that is not currently serving a request — which is why
`undefined_addr` is published rather than left for the reader to infer
structurally. The first tests the very pointer the next line dereferences, so it
needs no assumption about any other field.

As in OTEP 4947, readers SHOULD validate before trusting: a mis-stepped pointer
walk yields garbage at the same offsets. Checking `valid == 1` is the minimum;
sanity-checking `attrs-data-size` as well as `OrderedHashMap` bucket count being
a power of two are cheap additional guards.

### Interaction with Existing Functionality

* **OTEP 4947.** Additive. This proposal defines a second value of
  `threadlocal.schema_version` and a second discovery walk; the record format
  and its parsing are shared. A reader supporting both selects on
  `schema_version`.
* **OTEP 4719.** Additive, in the manner OTEP 4947 already established: four
  more `threadlocal.*` keys in `ProcessContext.attributes`.
* **OpenTelemetry SDKs.** Additive and optional. Nothing about existing Node.js
  SDK behaviour changes; a process that does not install the hook is
  indistinguishable from today.

### Runtime requirements

The mechanism requires `AsyncLocalStorage` to be backed by `AsyncContextFrame`,
since that is what puts the store map into the CPED slot. In Node.js this is
available from 22.7.0 behind `--experimental-async-context-frame`, and on by
default from Node 24, where it can still be turned off with
`--no-async-context-frame`.

An SDK MUST feature-detect this rather than infer it from the version and
command line, which disagree in both directions: `NODE_OPTIONS` can enable or
disable it without appearing in `process.execArgv`, and worker threads may be
created with a different `execArgv` than the main thread. Inferring "on" when it
is off is the dangerous direction — the SDK keeps working from JavaScript's
point of view while the CPED slot is never written, so every reader sees a
record that nothing updates. A direct probe (asking native code what is in the
CPED slot during a `run()`) tests the exact slot readers depend on.

An SDK that cannot satisfy the requirement MUST NOT publish
`threadlocal.schema_version`.

## Trade-offs and mitigations

### Dependence on V8's internal object layout

The reader walks `JSMap` and `OrderedHashMap`, neither of whose layouts is part
of V8's public API, and any of the offsets could change in a future V8.

**Mitigation:** the offsets are not hardcoded in readers. They are captured at
addon-compile time from the very V8 headers the addon is built against and
published through the process context, so a reader is told the layout of the V8
it is actually looking at. This does not protect against V8 restructuring these
objects more deeply than an offset change, which is what the versioned
`schema_version` is for. It does mean that the usual failure mode — a Node.js
release built with different pointer-compression or sandbox settings — is
handled without reader changes.

### Reader complexity relative to OTEP 4947

OTEP 4947's reader dereferences one thread-local to reach a record. This one
walks a hash map. That is more code, and it has to be defensive.

**Mitigation:** the identity hash narrows the search to one bucket, so the walk
is short in practice; and the two early exits mean the full walk only runs for
threads that actually have context attached. The complexity is confined to
reaching the record — everything from the record onward is shared with OTEP
4947.

### Rehashing of the map

Since `AsyncContextFrame` is a JavaScript `Map`, once can rightly ask what would
happen if it were mutated in place and triggered a rehash while it is being
read. Fortunately, the way it is currently implemented in Node.js is that
existing maps are never mutated, but instead they are copied on writes.

### Garbage collection

The objects on the walk (`AsyncContextFrame`, its backing table, the wrapper)
live in the V8 heap and can be moved by a garbage collection. OTEP 4947's
signal-handler model assumes the sampled thread is stopped, which prevents the
writer from racing the reader — but it does not by itself establish that no
*other* thread can relocate the objects being walked while the sampled thread is
stopped.

In practice, V8 performs object motion during the atomic pause on the isolate's
own thread, which is the stopped thread; concurrent GC threads mark rather than
move. We believe this makes the walk safe under the same assumptions OTEP 4947
already makes.

Note that the record itself is not a V8 heap object; it is malloc'd memory owned
by the wrapper, so it never moves as a result of GC. Only the path to it
involves heap objects.

Should that walk prove unsafe, a writer MAY close the gate for the duration of a
collection: register GC prologue and epilogue callbacks on the isolate, zero
`cped_slot` in the prologue and restore it in the epilogue, with the same
compiler fence and volatile store the other gate writes use. Readers need no
change at all as they already stop at a zero gate, and are forbidden from
treating it as permanent.

Losing the trace context for GC samples is a design decision. A collection is
triggered by whole-heap pressure that the active request may have contributed
little to, so attributing that time to whichever context happened to be current
would manufacture a plausible-looking but ultimately wrong attribution.

We are deliberately not going further at the moment. The obvious next step would
be capturing the active record's pointer in the prologue and publishing it in a
new field, so that samples during a collection keep their context. It would also
need a boolean to accompany it, because a zero field value cannot distinguish
"not collecting" from "collecting with no context attached", and the second
would send the reader down exactly the walk this is meant to prevent. It would
also have the writer perform a constrained version of the reader's walk from
inside a GC callback, where JS execution is prohibited, `GetCurrentContext()`
may be empty and so a `Global<Context>` must be retained solely for the lookup.
That is a lot of machinery to preserve an attribution we argue above should not
be made.

### Sampling a thread that is not executing JavaScript

A thread can be sampled while no JavaScript is on its stack at all: an event
loop with nothing to do is parked in the libuv poll. `cped_slot` addresses a
field of the isolate rather than anything on the JS stack, so the read itself is
unaffected. Node keeps an isolate entered for the whole lifetime of the event
loop it serves, both on the main thread and in worker threads, so the
not-entered state barely arises while an application is running. The only loop
that does spin with no isolate entered is the one a worker runs while it waits
for the platform to release its isolate during teardown, by which point our gate
is already closed.

When the loop is idle, the CPED slot holdswhatever frame was current at the
outermost level. Node unwinds the slot as the stack unwinds; every entry into
JavaScript goes through `InternalCallbackScope`, which exchanges the frame on
entry and restores the prior one on scope exit. Tick, timer and promise runners
do the same explicitly. Thus, an idle loop correctly does not retain the frame
of the request that last ran. It normally exposes `undefined`, which the
reader rejects by comparison against `undefined_addr`. The exception would be a
context installed with `enterWith` at the outermost level of the JavaScript
program and never cleared, which does persist. This is not a common practice,
and if it occurs, it could rightfully be considered the top-level context of the
program.

This is also mostly a wall-clock concern. A thread parked in the poll consumes
no CPU and so is never sampled by a CPU-time profiler.

We do not publish an "executing JavaScript" flag for readers to consult. A
reader that can walk the target's stack can already tell a thread parked in the
poll from one that is running, which is all such a flag would say; and
maintaining it would mean marking entry to and exit from JavaScript, which is
per-call work on the hottest path this design exists to keep native code off.

### Reader-visible consequences of incomplete teardown

An SDK that skips the teardown steps above does not endanger readers directly; a
cross-process read of freed or unmapped memory fails or yields garbage, it does
not fault the reader. The hazard is that a stale walk can still *succeed*. A
garbage word read through a dangling `cped_slot`, untagged as a `JSMap` and
walked, may reach an address whose byte 24 happens to be `1`, at which point the
reader publishes a fabricated trace ID and span ID attached to a genuine sample.
The probability is low; the consequence is corrupt telemetry that looks
legitimate, which is considerably harder to diagnose than a dropped sample. This
can happen in practice, because worker-isolate teardown happens while the
process continues to run and be sampled.

**Mitigation:** the MUST above, which closes the walk at its root for one store.
Readers should not rely on writers alone, though, which is why record validation
is recommended in the reading protocol: `valid == 1` is the minimum, and
checking `attrs-data-size` for plausibility and the `OrderedHashMap` bucket
count for being a power of two are cheap enough to be worth doing.

### Memory overhead

One record per live context, at most 640 bytes and typically 64, plus one small
JavaScript wrapper object and maybe some internal bookeeping structures of
approximately 40 bytes. An SDK caching wrappers per span holds them for the
span's lifetime. The thread-local struct is four words per thread.

### Trace sampling

Unchanged from OTEP 4947: an out-of-process reader cannot influence in-process
sampling decisions, so samples may reference traces the SDK never exported. The
same mitigation applies — publish the attributes that matter directly in
`attrs-data` via `attribute_key_map`.

## Prior art and alternatives

**Polar Signals' Node.js custom labels.** The approach of walking the
async-context frame out of the CPED slot was demonstrated by Polar Signals
([blog
post](https://www.polarsignals.com/blog/posts/2025/11/19/custom-labels-for-node-js)),
and OTEP 4947 already points at it as the likely direction for Node.js. This
proposal adopts that discovery idea and adapts it to OTEP 4947's record format
and OTEP 4719's process context.

**Writing the OTEP 4947 thread-local on every attach/detach.** Rejected: an FFI
crossing per context transition, on Node.js's hottest path, paid whether or not
a reader exists. This is the option OTEP 4947 already declined for Node.js.

**A native-side map keyed by async ID.** The SDK could maintain its own native
structure mapping async IDs to records and publish a pointer to it. Rejected: it
reintroduces a native call per transition to keep the map current, requires the
SDK to shadow bookkeeping the runtime already does, and gives readers a bespoke
structure to parse rather than a V8 one whose layout can be published.

**Publishing the record pointer in a JS-visible field instead of an internal
field.** Rejected: a JS-visible property is a tagged value subject to V8's
property-storage rules (in-object versus backing store, dictionary transitions),
so its location is neither stable nor cheaply computable by a reader. An
internal field is at a fixed offset and holds a raw aligned pointer.

**Pointing internal field 0 at a writer-side structure rather than the record.**
An earlier iteration of the prototype stored a pointer to the native wrapper
object and published its record-pointer offset as a fifth constant, so readers
made two hops. Rejected: it put a writer-implementation detail into the reader
contract for no benefit. Pointing directly at the record removed both the hop
and the published offset.

## Open questions

1. **GC and object motion.** Is the argument in "Garbage collection" above
   airtight across the V8 versions Node 22–26 ship, including parallel
   scavenging? Do we want to expose indication of GC going on (or even the
   current record during GC) in the thread local data structure?
2. **`attrs-data` and worker threads.** `attribute_key_map` is process-scoped
   per OTEP 4719, while records are per-isolate. Should the OTEP require all
   isolates in a process to agree on the key map (the simple reading, and what a
   single shared map implies), or is per-isolate divergence worth supporting?
3. **Should the V8 layout constants live in `threadlocal.*` at all?** They
   describe the runtime, not the thread-context mechanism, and a future
   non-profiling reader might want them too. A separate namespace (`v8.*`?)
   would be more honest but fragments the keys a reader of this schema must
   collect.
4. **Naming.** `threadlocal.*` is inherited from OTEP 4947, but nothing in this
   proposal is thread-local except the discovery struct. Keeping the prefix
   maximizes reuse of OTEP 4947's conventions and reader code; it is nonetheless
   a slight misnomer here.

## Prototypes

* **Writers:**
  * [`polarsignals/custom-labels`, `js/`
    directory](https://github.com/polarsignals/custom-labels/tree/otel-thread-ctx-wip/js)
    — reference implementation of this proposal, including the reader-facing
    contract documented in its `README.md`.
  * [`DataDog/pprof-nodejs`](https://github.com/DataDog/pprof-nodejs) — the same
    writer vendored into a shipping profiler, with tests covering the
    publication protocol, in-place and reallocating appends, invalidation, and
    isolate teardown.
* **Readers:** none yet implement this walk. The [ctx-sharing-demo
  reader](https://github.com/scottgerring/ctx-sharing-demo/tree/main/context-reader)
  and the [eBPF profiler
  PR](https://github.com/open-telemetry/opentelemetry-ebpf-profiler/pull/1229)
  implement OTEP 4947's walk and would share all record parsing with this one.

## Future possibilities

* **Other runtimes with the same shape.** The pattern — publish a discovery
  struct once, let the runtime do the context switching, teach the reader to
  walk runtime internals — should transfer to other managed runtimes whose
  context model is not the OS thread.
* **Sharing the V8 layout constants.** If the constants outgrow this mechanism,
  they could be promoted out of `threadlocal.*` and reused by any reader that
  needs to walk a V8 heap.
* **Non-Linux platforms.** As with OTEPs 4719 and 4947, the discovery contract
  here is ELF/TLSDESC-based. The record format and the CPED walk are not
  Linux-specific; only the mechanism for finding the discovery struct is.
