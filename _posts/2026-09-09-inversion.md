---
layout: post
title: "What WebKit fixes leave behind: six cases in variant auditing"
image: /img/webkit.svg
tags: [webkit]
---
* TOC
{:toc}

Some WebKit fixes expose their own variants. A patch closes one route to an attacker-usable primitive—an out-of-bounds read, an unauthorized cross-origin access—but a sibling path in the same code area may still reach the same bad state. Sometimes the fix itself adds an operation with a new gap. In other cases, it extends an existing mechanism to cover one missed case, raising the question of what else remains uncovered.

Fixing the reported bug and auditing its variants are different tasks. A patch author is solving the specific failure in front of them. Enumerating every other path to the same primitive is a separate exercise, and it often gets deferred.

[WebKit Weekly](https://webkitweekly.com/) publishes per-commit security analyses of WebKit fixes each week. Reading the archive for fixes that exposed related gaps, six cases stood out from the last few months. They fall into three patterns, summarized in the taxonomy at the end.

---

# Pattern 1 — invalidation coverage gaps

JSC caches compiled code, resolved callee addresses, and refcounted data retained for concurrent readers. Each cache is valid only while the state it mirrors remains unchanged. When code is detached, a GC phase ends, or a reader finishes, the corresponding cached or retained state has to be cleared. Miss one of those relationships and a later access can reach freed data or detached code.

That invalidation work is spread across many functions in the runtime, each triggered by a specific event and each responsible for clearing a specific set of caches. When someone adds a new cache without wiring it into the existing invalidation code, or adds a new invalidation operation without covering every state the runtime can be in, the coverage matrix ends up with holes. The two cases below expose opposite holes in that matrix: Case A added a new clearing operation without accounting for every active reader, while Case B added a new cache without connecting it to an existing invalidation operation.

## Case A — W23 GC concurrent-retained data pair

- [`e69c479`](https://github.com/WebKit/WebKit/commit/e69c47917811c2d01befd9e16205c756c86e06a6) : JSString use-after-free via `GCOwnedDataScope` and atomization swap. [W23 report](https://webkitweekly.com/report/2026-W23/commits/e69c479178).
- [`c8e53c7`](https://github.com/WebKit/WebKit/commit/c8e53c7440c6c74d1c2fa76b7a6a30fd44ac3706) : `Heap::clearConcurrentRetainedDataIfPossible()` must not run while concurrent marking is active. [W23 report](https://webkitweekly.com/report/2026-W23/commits/c8e53c7440).

A `JSString` holds its actual character bytes in a companion `StringImpl`. `StringImpl` is refcounted. When its last reference goes away, the destructor runs and the memory is freed immediately. Native string operations get raw views into these buffers via `GCOwnedDataScope`, a stack-scoped guard that pins the owning `JSString`. Atomization is JSC's string interning: when a `JSString` is used as a property key and a canonical atom already exists, the engine replaces the string's backing `StringImpl` with the shared one and moves the old impl's last strong reference into a per-VM retained list.

The key mismatch was that `GCOwnedDataScope` kept the `JSString` wrapper alive, while the native operation held a raw view into its backing `StringImpl`. During argument coercion, attacker-controlled script could re-enter JSC and atomize the same string. Atomization replaced the `JSString`'s backing impl with a canonical one. The native operation's raw view still addressed the old impl's buffer, but the retained list now held its only remaining strong reference. If the callback triggered a GC, `Heap::finalize()` cleared that list and freed the old impl. The `JSString` remained alive throughout, but the buffer the native operation was about to read did not. When execution returned to the native operation, its raw view was dangling.

The fix stopped treating the retained list as something that could be cleared wholesale at the end of every GC. It paired each old `StringImpl` with the `JSString` that had previously owned it, then used conservative stack scanning to determine which owners were still live. If an owner was still visible on the stack, its old impl remained in the list. This kept the buffer alive across a GC when a native scope might still be using it.

The tradeoff was that entries could now survive longer and accumulate between collections. To control that growth, the same patch added a separate drain through the incremental sweeper, intended to clear the list when none of the known readers was active.

That drain had to account for another class of reader: concurrent GC markers. JSC's garbage collector runs partly concurrently with the JS thread. Marker threads walk the heap graph and dereference raw pointers without taking a reference on every access. Any object a marker is inspecting therefore has to remain alive until the marking pass finishes.

The new drain missed that case. `clearConcurrentRetainedDataIfPossible()` avoided clearing while JavaScript was executing or JIT compilation was in flight, but it did not check whether concurrent markers were active. If the sweeper fired during marking, it could drop the retained reference to a `StringImpl` that a marker thread was still reading and free the object underneath it. The sibling patch added the missing `mutatorShouldBeFenced()` guard.

When a fix adds a new operation that runs concurrently (like a sweeper timer), that operation starts with a list of `if` conditions checking whether it's safe to run right now. Whoever wrote the list wrote down the cases they thought of. Some cases get missed. When you review a fix that adds a new concurrent operation, that safety-check list is the first place to read.

## Case B — W33 MicrotaskCallCache retains detached CodeBlock entry points

- [`75a9d41`](https://github.com/WebKit/WebKit/commit/75a9d414a4a8b98d26840162d103eb16227eaeb6) : JSC MicrotaskCallCache retains detached CodeBlock entry points. [W33 report](https://webkitweekly.com/report/2026-W33/commits/75a9d414a4).

JSC compiles hot JS functions to native machine code (JIT compilation). Each piece of JS source has a persistent `ScriptExecutable` handle, and when the JIT compiles it, the result (a `CodeBlock`) attaches to that executable. JSC also caches the entry-point address of the compiled body at each call site so repeat calls jump straight there instead of resolving the callee again.

`Heap::deleteAllCodeBlocks()` is a proactive invalidation mechanism that detaches compiled `CodeBlock`s from their `ScriptExecutable`s. Any independent cache that stores their entry-point addresses has to be cleared as part of the same operation.

`VM::m_syncResumeCallCache`, a per-VM cache for the async-generator microtask path, was introduced five weeks earlier in [`bafc1f2`](https://github.com/WebKit/WebKit/commit/bafc1f2d7f1e8c6c2eb2ebc395c8f94af5f35c33), but `deleteAllCodeBlocks()` was not updated to clear it. After the detach, the still-live ScriptExecutable could continue to satisfy the cache's lookup key even though the cached CodeBlock was no longer the code installed on that executable. A later microtask resume could therefore reuse an entry point that `deleteAllCodeBlocks()` had intended to retire.

The fix adds one line, `vm.clearMicrotaskCallCaches()`, to `deleteAllCodeBlocks()`. The commit message notes that Wasm's equivalent cache was already cleared there. The patch applies the same invalidation rule to the JS-side cache.

Case A and Case B expose opposite sides of the same matrix. Case A added an invalidation operation but missed one category of active reader. Case B added a cache but missed one invalidation trigger. A new cache should be checked against every existing invalidation path, and a new invalidation path against every cache it is expected to clear.

---

# Pattern 2 — the fix missed sibling code paths

A patch modifies one function, one opcode, one call site. Structurally identical siblings in the same file didn't get touched.

Three families here, sitting at different ends of a timing spectrum. Widening had three sibling fixes land in the same week. table64 took two weeks. The WebGL PBO offset pair took eleven.

## Case C — W34 Wasm validator widening family

Four commits in W34, all in `Source/JavaScriptCore/wasm/WasmFunctionParser.h`, all fixing the same underlying issue at different opcodes (listed chronologically):

- [`8f229fb`](https://github.com/WebKit/WebKit/commit/8f229fb72961093d2f552ecfef65a8f13afd402a) : WebAssembly try/catch result-type confusion omits optimized reference checks. [W34 report](https://webkitweekly.com/report/2026-W34/commits/8f229fb729).
- [`36058b8`](https://github.com/WebKit/WebKit/commit/36058b8e69ac846ca93e33feb106b09d4b2fd787) : Unreachable end ops don't widen result types. [W34 report](https://webkitweekly.com/report/2026-W34/commits/36058b8e69).
- [`261bcb4`](https://github.com/WebKit/WebKit/commit/261bcb42d83f2838cb8245675ffd6e53eebf55bc) : Delegate should widen types like End. [W34 report](https://webkitweekly.com/report/2026-W34/commits/261bcb42d8).
- [`9dcbd25`](https://github.com/WebKit/WebKit/commit/9dcbd254af217b64f37947d680ce0c6eae0ab7e3) : Argument and Result block types should always widen. [W34 report](https://webkitweekly.com/report/2026-W34/commits/9dcbd254af).

WebAssembly is validated before it runs. The validator type-checks each instruction against an abstract operand stack, tracking what type each value would have. Structured blocks such as if/else and try/catch can produce values through multiple control-flow paths. When those paths converge, the parser has to record the block's declared result type, rather than the narrower type left behind by whichever arm it parsed last.

If widening is skipped, the recorded type is narrower than what the runtime can actually deliver at that merge. For instance, if one branch pushes a specific struct reference and another pushes any reference at all, the merged type should widen to cover both but is left as the narrow specific type. JIT tiers trust this recorded type and compile away the runtime checks that would have caught the mismatch. The result at runtime is a type confusion. Whatever value came from the wider branch gets treated as if it had the narrow type, and downstream operations interpret it according to the wrong structural assumptions.

The first of the four (`8f229fb`, try/catch) is the seed. Its commit message notes that widening had previously only covered if/else, and this patch expanded it to try/catch. The other three commits, landing within 26 hours, fix the sibling sites the seed still didn't reach: the `End` handler in the parser's path for unreachable code, the Delegate opcode, and Argument/Result block types.

When an operation is implemented in multiple places, fixing one doesn't fix the others. `WasmFunctionParser` implements `End` in two separate functions: one for normal parsing, and one for parsing code that appears after `br` or `throw` and can't actually execute at runtime. Several other opcodes in the same file also merge control flow. The seed patched one merge site. The three W34 siblings patched three additional sites with the same widening requirement.

## Case D — Wasm table64 migration gaps

Three commits over two weeks fixing separate gaps in the table64 width transition:

- [`862994e`](https://github.com/WebKit/WebKit/commit/862994e2cc9c67df80c6ef6f36c73fbbd7e7614a) : Wasm table imports don't check that the address type matches. [W30 report](https://webkitweekly.com/report/2026-W30/commits/862994e2cc).
- [`b91045c`](https://github.com/WebKit/WebKit/commit/b91045c99b8a7c68df9d3f5b8e08e51a5f97a2f4) : Do not truncate table64 maximum size to uint32_t. [W31 report](https://webkitweekly.com/report/2026-W31/commits/b91045c99b).
- [`e942b93`](https://github.com/WebKit/WebKit/commit/e942b93cda98c85f8c15f5add8ef8ecb85f5d90a) : Active element segment offsets are truncated for a table64. [W31 report](https://webkitweekly.com/report/2026-W31/commits/e942b93cda).

Wasm's `Memory64` proposal allows memories beyond 4 GiB, and `table64` extends table indices beyond 2^32 entries. Supporting a wider address type requires every stage that stores, validates, or consumes it to preserve the width. A remaining `uint32_t` field can truncate the value, while an import check that ignores address type can connect code compiled for one width to a table using another.

W30's `862994e` handles a missing address-type check on table imports. If a module imported a table whose address type didn't match the module's own declaration, the import wasn't rejected. W31 adds two more: `b91045c` fixes the `maximum` size field being stored as `uint32_t`, and `e942b93` fixes the offset of an active element segment truncating when the target table is 64-bit. Three sites in the same subsystem, each one a spot the original 32→64 extension had missed.

Width migrations are therefore best audited end to end: import validation, metadata storage, constant-expression evaluation, initialization, and JIT bounds assumptions all have to agree on the same width.

## Case E — WebGL PBO offset reinterpret pair

Two commits eleven weeks apart, both in `Source/WebCore/platform/graphics/angle/GraphicsContextGLANGLE.cpp`, both fixing the same anti-pattern at sibling GL entry points:

- [`d0000ca`](https://github.com/WebKit/WebKit/commit/d0000cab59a2bb2ae9dbb8afa66920ff2c2f59ff) : WebGL `readPixels` PBO offset reinterpreted as a host pointer. [W24 report](https://webkitweekly.com/report/2026-W24/commits/d0000cab59).
- [`487e5a0`](https://github.com/WebKit/WebKit/commit/487e5a08ac6cf4bc87660eaebc2e05e72bf0504b) : WebGL: Reject offset-based GL calls when no buffer is bound. [W35 report](https://webkitweekly.com/report/2026-W35/commits/487e5a08ac).

In WebKit's GPU-process configuration, the WebContent process sends WebGL calls across IPC to the GPU process, which replays them through ANGLE, the graphics translation layer that turns OpenGL calls into Metal on Cocoa platforms.

OpenGL ES 3 overloads a single parameter to mean either a host memory address or a byte offset into a GPU buffer, depending on binding state. `readPixels` takes a `void*` destination that is a host pointer when no *pixel pack buffer* (PBO) is bound and a byte offset into the PBO when one is. `tex(Sub)Image` and `compressedTex(Sub)Image` have the equivalent overload for their `pixels` source parameter, driven by `GL_PIXEL_UNPACK_BUFFER` binding. Same shape across all these entry points.

Both commits expose the same underlying ambiguity: a numeric GL argument crossing between buffer-offset and host-pointer interpretations. But the two sit at different layers. In W24, GL correctly treated the value as a bound-PBO offset (a PBO was in fact bound), but WebKit's shared post-processing path (`wipeAlphaChannelFromPixels`) treated the same value as a host address and wrote through it. In W35, with no unpack buffer bound, ANGLE itself treated the IPC-supplied offset as a client pointer and read the texture data from that address. The WebContent-side WebGL bindings normally reject that call, but a compromised WebContent process can emit the IPC message directly and skip the check.

The primitives differ. W24's write is bounded by allocatable PBO sizes, making a deterministic low-address crash more realistic than a useful write primitive. W35 provides a GPU-process arbitrary-address read into a sampleable texture. The compromised WebContent process can then read that texture back through the normal WebGL path, turning the raw read into a memory-disclosure primitive.

Same file, sibling GL entry points, same underlying offset/pointer ambiguity at different layers. Eleven weeks is a long time for a sibling path in the same file to remain unfixed once the pattern is visible in the diff.

---

# Pattern 3 — same shape at sibling subsystems

The patch adds an authorization or validation pattern at one subsystem. Other subsystems in the same file (or same architectural layer) have the same identifier/handle shape and needed the same treatment.

## Case F — NetworkStorageManager identifier ownership family

Four commits over about three months, all touching `Source/WebKit/NetworkProcess/storage/NetworkStorageManager.cpp`:

- [`d854553`](https://github.com/WebKit/WebKit/commit/d85455322dae47dcb4235d0ca0dce0fee75a0fb5) : IndexedDB Connection/Transaction Identifier Confusion. [W21 report](https://webkitweekly.com/report/2026-W21/commits/d85455322d).
- [`07d83d0`](https://github.com/WebKit/WebKit/commit/07d83d0d699eedc4e80908d99470a5fca06acdc8) : Validate connection access to FileSystem storage with `FileSystemHandleIdentifier`. [W32 report](https://webkitweekly.com/report/2026-W32/commits/07d83d0d69).
- [`5d1be2c`](https://github.com/WebKit/WebKit/commit/5d1be2cedef6f18889a7c6bb55899aed91c7f16f) : Validate connection access to DOMCache with `DOMCacheIdentifier`. [W32 report](https://webkitweekly.com/report/2026-W32/commits/5d1be2cede).
- [`df8786e`](https://github.com/WebKit/WebKit/commit/df8786e8b6787d81b5f96e0b91cbc1a67d5cf7fc) : Enable storage site validation. [W33 report](https://webkitweekly.com/report/2026-W33/commits/df8786e8b6).

WebKit splits storage work across processes. WebContent runs untrusted JavaScript, while the network process owns persistent storage and services requests for IndexedDB, FileSystem, and CacheStorage through `NetworkStorageManager`.

These subsystems refer to open resources through identifiers such as `IDBResourceIdentifier`, `FileSystemHandleIdentifier`, and `DOMCacheIdentifier`. The identifiers behave like capabilities, but cross IPC as attacker-controlled integers. Their registries can resolve resources created through multiple WebContent connections, so finding an identifier in a registry does not by itself prove that the sender owns it.

A compromised WebContent process could therefore submit an identifier associated with another connection or origin, provided it could obtain or predict the value. Without an additional ownership check, the network process would resolve the identifier and act on the resource as though the sender were authorized.

The concrete fixes differed. IDB bound resource identifiers to the sending IPC connection at the registry choke point. FileSystem and DOMCache recovered the resource's owning site and checked whether that site was allowed for the sender via `isSiteAllowedForConnection`. The shared invariant was sender-to-resource ownership, not a uniform lookup implementation.

W21 fixes IDB. A broader sibling audit followed twelve weeks later, led by a different author through two same-day commits covering FileSystem and DOMCache. That same author enables site validation by default the following week.

The missing-ownership-check pattern remained in FileSystem and DOMCache for twelve weeks after the IDB fix. Once the broader audit began, three commits followed within a week.

---

# When the file itself is the signal

Three reports touch `Source/JavaScriptCore/heap/Heap.cpp` over ten weeks: the W23 pair from Pattern 1, plus the W33 MicrotaskCallCache fix from the same pattern. All three sit in JSC's lifetime and invalidation machinery, although they involve different consumers: retained GC data in W23 and detached JIT code in W33. `NetworkStorageManager.cpp` from Pattern 3 shows the same signal from a different subsystem, with four fixes in a three-month window. `GraphicsContextGLANGLE.cpp` from Case E shows it with just two, both touching the same file with related bug shapes.

Repeated security fixes in the same file are a useful prioritization signal. They do not identify a specific bug, but they show where patterns from earlier fixes may be worth applying again.

---

# The three shapes

Invalidation coverage gaps. A cache doesn't get invalidated by the mechanism that should reach it, or an invalidation operation itself has a race in its guard list. The tell: a diff that adds one line calling a new invalidation function into an existing invalidation pass, or adds a guard clause to a periodic clearing routine. Both mean the invalidation matrix had a hole. Ask which other caches or which other invalidation triggers might have the same missing edge.

Fix missed sibling code paths. The patch modifies one instance of a class of similar constructs (opcodes, message handlers, decoder call sites, GL entry points that share a parameter shape). Search for structurally similar sites and compare them with the sites touched by the diff. Any untouched matches form a candidate list. This pattern is the easiest for a fix author to turn into an in-repo sweep before shipping. The timing shows whether that sweep actually happened.

Cross-subsystem shape. The patch adds an authorization or validation pattern at one subsystem, and the file (or the surrounding directory) contains sibling subsystems with the same identifier/handle shape. The tell: a new call to an origin check or capability lookup added to one message handler, with structurally similar handlers in the same file untouched. Ask which other resource types in the same file received the same treatment. Same-file clustering is often what surfaces the family in the first place.

---

# Closing

Across the six cases, what matters is not primitive severity, but how long the broader family went unaudited after the pattern first became visible.

W34 Wasm widening had three sibling fixes land in the same week. W23 GC pair, same week. Wasm table64 took two weeks between the first fix and the two further table64 gaps. MicrotaskCallCache existed for five weeks unwired before `deleteAllCodeBlocks()` was updated to clear it. WebGL PBO offset took eleven weeks between the two commits fixing the same `reinterpret_cast` anti-pattern at sibling GL entry points. NetworkStorageManager took twelve weeks between the IDB fix and the sibling subsystems.

Every gap between a fix that exposes a pattern and its family audit is a window during which the pattern is visible to anyone reading the diff. The number matters less than the shape. Same-week clusters suggest that the patch expanded into a broader audit. Cross-week clusters show what can remain exposed when that expansion does not happen immediately.

When a fix lands, diff readers should ask one question: which sibling paths, if any, remain unaudited?

If you want to try this on the next few months of WebKit fixes, [WebKit Weekly](https://webkitweekly.com/) is where the per-commit reports live.
