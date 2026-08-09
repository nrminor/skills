---
name: performance-design
description: Design software so its architecture preserves a high performance ceiling before implementation. Use proactively during architecture discussions, data-model and API design, implementation planning, and code sketching—even when optimization was not explicitly requested—especially for repeated computation, data-heavy paths, long-lived services, libraries, or choices expensive to reverse. Examines data layout, indirection, allocation topology, cache locality, branch predictability, SIMD, batching, and data movement; also use after implementation to distinguish local tuning from representation rewrites.
---

# Performance Design

Treat performance as a design responsibility. Modern software routinely spends
far more CPU time, memory, energy, battery, cloud capacity, and hardware than
its work inherently requires. Preventing that waste is materially useful, and
usually easier than recovering performance after interfaces and representations
have hardened.

The goal is **non-pessimization**: preserve a high performance ceiling without
introducing speculative complexity. Reason from workload shape and hardware
costs before implementation. Use benchmarks and profilers later to choose among
viable designs and tune them; measurement cannot make a pointer jungle
contiguous or restore opportunities an API has already ruled out.

Design is the highest-leverage performance phase. Profiling begins after many
choices about representation, boundaries, and ownership have hardened; use it
to refine those choices rather than postponing all performance thought until
they are expensive to reverse.

## Start with the workload

Establish rough bounds rather than demanding a completed implementation:

- items, bytes, or requests processed
- operation frequency and expected growth
- latency, throughput, and memory constraints
- target hardware and deployment environment
- read/write ratio and access patterns
- whether work is interactive, batched, streaming, or offline

Order-of-magnitude estimates are enough to expose important design differences.
If the workload is genuinely tiny, cold, and cheap to change, say so and keep
the design simple. Spend complexity only where the workload or irreversibility
justifies it.

Sketch the four physical resources and both characteristic costs for each:

| Resource | Bandwidth question                    | Latency question                                     |
| -------- | ------------------------------------- | ---------------------------------------------------- |
| Network  | How many bytes cross the boundary?    | How many round trips occur?                          |
| Disk     | How much data is transferred?         | How many seeks, syncs, or operations occur?          |
| Memory   | How large is the working set?         | How much indirection and cache-miss exposure exists? |
| CPU      | How much useful work occurs per item? | How predictable are branches and dependencies?       |

Frequency can reverse the apparent ordering: a small memory penalty repeated
billions of times may dominate a visibly slower resource used once.

## Tier 1: Preserve the performance ceiling

Audit representations before polishing APIs or implementation details.

### Representation

- Compare contiguous storage with separately allocated objects and pointer
  traversal.
- Compare array-of-structs with struct-of-arrays based on how fields are read.
- Prefer compact IDs or dense indices when stable object identity is unnecessary.
- Separate hot fields from cold metadata so the active working set stays small.
- Look for object-per-row or object-per-element models that could instead be
  columnar, packed, or represented as a few large buffers.
- Account for object headers, pointers, alignment, capacity slack, and allocator
  metadata—not merely payload bytes.

### Data movement

- Trace bytes across parsing, decoding, copying, serialization, IPC, FFI, and
  network boundaries.
- Challenge materialized intermediates and repeated format conversions.
- Prefer views, slices, batches, and ownership transfer when their lifetimes are
  safe and legible.
- Treat movement through the memory hierarchy as work; arithmetic is often the
  cheap part.
- Batch network messages, disk operations, memory traversal, and CPU work. Do
  not restrict batching to loops while leaving every other boundary chatty.

### Iteration and control flow

- Prefer dense, predictable iteration over scattered callbacks and virtual or
  dynamic dispatch in inner loops.
- Identify data-dependent branches that may be reorganized, partitioned, or
  converted into branch-light batch operations.
- Preserve regular loops that compilers, native libraries, or runtimes can
  vectorize with SIMD.
- Avoid API seams that require one-item-at-a-time processing when callers could
  provide a batch.

### Lifetimes and allocation topology

- Distinguish many small allocations from a few large allocations. Heap use is
  not itself the problem; independent objects, indirection, bookkeeping, and
  poor locality often are.
- Group values with the same lifetime. Consider arenas or bump allocation when
  whole lifetime cohorts can be released together.
- Reject arenas when they would retain unrelated data, obscure resource cleanup,
  or fight the ownership model.
- Preserve opportunities for scratch-buffer reuse and bounded-capacity storage.
- Prefer stack, inline, static, or startup allocation when bounds are modest and
  lifetimes make that choice safe. Treat this as a design bias, not an absolute:
  large or unbounded values belong in appropriately managed dynamic storage.
- Put explicit upper bounds on queues, batches, retries, and work units whenever
  the domain permits it. Bounds improve capacity reasoning and prevent hidden
  tail-latency cliffs.

### Boundaries and concurrency

- Scrutinize public data models and library APIs more heavily than private code:
  callers build architecture on top of them, making representation mistakes
  expensive to reverse.
- Partition ownership so workers operate on independent, contiguous regions
  where possible.
- Account for synchronization, false sharing, queueing, and cross-thread data
  movement before proposing more concurrency.
- Separate the control plane from the data plane. Let the control plane perform
  rich validation, scheduling, and policy decisions; give the data plane large,
  predictable, already-validated units of work.

## Sketch competing representations

Produce at least two small code sketches when a consequential representation
choice remains open. Show internals, callsites, and what the API teaches callers.

```rust
// Object graph: ergonomic, independently allocated, traversal-heavy.
struct Node {
    next: Option<Box<Node>>,
    position: Position,
    label: String,
}

// Dense storage: compact IDs, separable hot and cold data, linear traversal.
struct NodeId(u32);

struct Nodes {
    next: Vec<Option<NodeId>>,
    positions: Vec<Position>,
    labels: Vec<String>,
}
```

Compare the sketches against the actual access pattern. Do not assume the dense
version wins merely because it looks data-oriented; explain which bytes each
operation touches and which future optimizations each representation preserves.

## Classify the local optimization ceiling

State one of these conclusions plainly:

- **Preserved** — the architecture permits compact layouts, batching, predictable
  traversal, and meaningful future tuning.
- **Constrained** — named seams or representations block some important options,
  but targeted redesign remains possible.
- **Low** — step-change improvement requires replacing core representations,
  ownership, or APIs; local tuning can provide only incremental gains.

A low local ceiling is not an irreversible point of no return. It means the
honest recommendation is an architectural rewrite rather than a long list of
micro-optimizations.

## Tier 2: Plan mechanically sympathetic implementation

Turn the selected architecture into explicit invariants:

- hot paths iterate over dense, homogeneous data
- hot and cold fields remain separated
- capacities and lifetime groups are known where practical
- scratch storage is reusable
- APIs accept views or batches without forced copies
- representations remain stable enough for runtime specialization
- native or vectorized execution remains possible
- ownership boundaries avoid redundant movement
- hot loops are isolated enough for both the compiler and a human to see their
  primitive inputs, memory access, branches, and redundant work

Prefer hot kernels with primitive arguments and little hidden state over methods
that make the compiler repeatedly prove facts about a large object graph. Keep
coordination outside the kernel and dense iteration inside it.

Use high-quality dependencies when they provide a better implementation than
the project should reasonably build itself—especially mature native,
vectorized, parsing, compression, and storage libraries. Evaluate dependencies
for representation conversions, forced copies, abstraction barriers, install
cost, and maintenance quality rather than treating dependency count as the goal.

Language constraints change the available mechanisms, not the objective:

- **Rust:** contiguous collections, slices, dense IDs, arenas for coherent
  lifetimes, compact enums, and explicit ownership make layout choices visible.
- **JavaScript/TypeScript:** favor packed arrays, TypedArrays, stable object
  shapes, batching, and transferable buffers over object-per-element models.
- **Python:** move repeated work onto compact storage and native loops through
  NumPy, Polars, Arrow, `array`, or `memoryview`; avoid Python-object-per-row
  processing when scale matters.

## Encode the design with assertions

Assertions are executable documentation of the cost model and representation.
Keep them in production when they defend programmer invariants; discovering a
violated assumption early hardens the application faster than allowing corrupt
state to travel deeper into the system.

- Assert capacities, indexes, batch bounds, representation invariants, and
  relationships among constants.
- Use compile-time assertions for type sizes, alignment, capacities, and other
  properties known during compilation.
- Validate batches at boundaries so the hot kernel can operate on a strong,
  explicit contract rather than rechecking every element through scattered
  control flow.
- Assert important invariants from more than one independent path when practical,
  such as immediately before serialization and after deserialization.
- Cover both expected valid states and the boundaries where valid data becomes
  invalid.

Assertions support understanding; they do not replace it. Write down why the
invariant matters and design tests that deliberately exercise its edges.

## Tier 3: Validate, tune, or salvage

After implementation, benchmark representative workloads and profile time,
allocations, cache behavior, branches, and data movement as appropriate.

1. Check whether measurements agree with the original cost model.
2. Compare candidate representations when architectural choices remain open.
3. Tune a viable design without erasing the properties that made it viable.
4. When the local ceiling is low, separate rewrite-scale recommendations from
   incremental salvage.
5. Load the **allocations** skill for allocation profiling, buffer reuse, clone
   reduction, pooling, and Rust-, V8-, or Python-specific local improvements.

Profiling is decisive for prioritizing local work. It is not a substitute for
asking whether the current representation made the desired performance
physically plausible.

## Review output

Scale the depth to the decision, but cover this shape for consequential designs:

```markdown
## Performance design review

### Workload shape

Rough scale, frequency, constraints, and access pattern.

### Candidate representations

At least two sketches and the bytes/control flow each implies.

### Performance ceiling

Preserved, constrained, or low—with the specific reason.

### Recommended design

Representation, ownership, batching, and boundary choices.

### Implementation invariants

Properties the implementation must retain.

### Validation plan

Representative benchmarks and post-implementation profiling questions.
```

Favor simple designs with favorable mechanical properties. A flat vector and a
straight loop are often both faster and easier to understand than a hierarchy
of tiny objects. Complexity that cannot be justified by workload shape is not
performance design; it is another form of pessimization.

## Influences

- [TigerStyle](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md)
  — especially its emphasis on upfront performance sketches, mechanical
  sympathy, bounded work, batching, assertions, and simple designs that advance
  safety, performance, and developer experience together.
