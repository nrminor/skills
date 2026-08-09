---
name: allocations
description: Audit and reduce heap/dynamic allocations in Rust, TypeScript/JavaScript, and Python after architecture is fixed or measurements identify allocation pressure. Covers allocation profiling, buffer reuse, clone reduction, pooling, stack and inline storage, zero-copy patterns, and runtime-specific tuning. For pre-implementation architecture, data layout, batching, cache locality, SIMD, data movement, or performance ceilings, load performance-design first.
---

# Heap Allocation Audit Skill

Guidance for identifying and reducing unnecessary heap allocations across Rust,
TypeScript/JavaScript, and Python. This is a local tuning and salvage skill.

> [!IMPORTANT]
> Thinking about allocations alone only gets you so far. If data representation,
> ownership, API shape, or processing architecture can still change, load
> **performance-design** first. A pointer-heavy or object-per-item architecture
> may have a low local optimization ceiling: fewer clones and better buffer reuse
> help, but step-change gains require a bigger-picture redesign.

## General Principles

1. **Classify the scope.** Use **performance-design** while architecture is
   malleable; use this skill when doing local tuning inside an accepted design.
2. **Measure local work.** Profile allocations and benchmark representative hot
   paths before choosing among local fixes.
3. **Allocation topology matters.** A few large allocations usually behave very
   differently from many independently managed objects.
4. **Cache locality matters.** Scattered allocations cause indirection and cache
   misses beyond allocator bookkeeping alone.
5. **Dependencies have costs.** Adding a crate or package for one specialized
   container may not earn its maintenance burden.

## Detailed References

For comprehensive patterns, examples, and profiling tool guides:

- **Rust**: See [references/rust-allocations.md](references/rust-allocations.md)
- **TypeScript/JavaScript**: See [references/typescript-allocations.md](references/typescript-allocations.md)
- **Python**: See [references/python-allocations.md](references/python-allocations.md)

---

# Quick Reference: Rust

## Heap-Allocated Types

| Type             | Allocates       | Notes             |
| ---------------- | --------------- | ----------------- |
| `Box<T>`         | Always          | Single allocation |
| `Vec<T>`         | On first push   | Grows: 0→4→8→16   |
| `String`         | On first push   | Same as Vec       |
| `HashMap<K,V>`   | On first insert | ~7 slots minimum  |
| `Rc<T>`/`Arc<T>` | On creation     | Refcount + T      |

## Quick Wins

- **`Vec::with_capacity(n)`** — Pre-allocate when size known
- **`clone_from(&b)`** — Reuse allocation instead of `a = b.clone()`
- **`buf.clear()`** — Reuse collections in loops
- **`&str` not `String`** — Accept references in function params
- **`Cow<'static, str>`** — Mixed static/dynamic strings
- **Avoid `format!()` for static strings** — Use `.into()` instead

## Anti-Patterns

| Pattern                            | Fix                     |
| ---------------------------------- | ----------------------- |
| `.clone()` in hot loops            | Use references or `Cow` |
| `.to_string()` on literals         | Use `&'static str`      |
| `collect::<Vec<_>>()` mid-iterator | Keep as iterator        |
| `Box<Vec<T>>`                      | Just use `Vec<T>`       |

## Profiling Quick-Start

```bash
# DHAT (Rust crate) - add dhat dependency, run with feature flag
cargo run --release --features dhat-heap
# View at: https://nnethercote.github.io/dh_view/dh_view.html

# Print type sizes
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release
```

---

# Quick Reference: TypeScript/JavaScript

## V8 Essentials

- **Hidden Classes**: Objects with same properties in same order share them
- **Element Kinds**: Arrays transition PACKED_SMI → DOUBLE → ELEMENTS (can't go back)
- **Inline Caches**: Monomorphic (1 shape) fastest; megamorphic (>4) slowest

## Quick Wins

- **Avoid array holes** — Use literals `['a','b']` not `new Array(2)`
- **Consistent object shapes** — Always initialize all properties
- **Don't `delete` properties** — Set to `undefined` instead
- **Use rest params** — Not `arguments` object
- **`JSON.parse`** — Faster than object literals for >10kB

## Anti-Patterns

| Pattern                    | Fix                     |
| -------------------------- | ----------------------- |
| Array holes                | Use literals or push    |
| `delete obj.prop`          | `obj.prop = undefined`  |
| Closures in hot loops      | Define function outside |
| `.map().filter().reduce()` | Single-pass loop        |
| String `+=` in loops       | Array + join            |

## Profiling Quick-Start

```bash
# Chrome DevTools: Memory tab → Heap Snapshot or Allocation Timeline

# Node.js
node --inspect index.js
node --heapsnapshot-signal=SIGUSR2 index.js  # Then: kill -USR2 <pid>
```

---

# Quick Reference: Python

## Quick Wins

- **`set` for membership** — O(1) vs O(n) for lists
- **`__slots__`** — Eliminates per-instance `__dict__`
- **`math.sqrt(n)`** — Faster than `n ** 0.5`
- **Pre-allocate lists** — `[0] * n` or list comprehension
- **Local variables in loops** — Cache globals locally
- **`itertools`** — C-optimized combinatorics
- **`bisect`** — O(log n) sorted list operations
- **`memoryview`** — Zero-copy slicing

## Anti-Patterns

| Pattern                 | Fix              |
| ----------------------- | ---------------- |
| `x in list`             | Use `set`        |
| No `__slots__`          | Add `__slots__`  |
| Exceptions in hot loops | Use conditionals |
| Nested loops for combos | Use `itertools`  |
| Slicing large buffers   | Use `memoryview` |

## Profiling Quick-Start

```python
# tracemalloc (built-in)
import tracemalloc
tracemalloc.start()
# ... code ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```

```bash
# memory_profiler
pip install memory_profiler
python -m memory_profiler script.py
```

---

# Audit Checklist

1. **Classify the problem** — Architectural performance ceiling or local
   allocation tuning? Load **performance-design** for the former.
2. **Identify hot paths** — Where does execution time concentrate?
3. **Profile allocations** — Use language-appropriate tools.
4. **Categorize**:
   - Necessary (can't avoid)
   - Reducible (reuse, pool, pre-allocate)
   - Eliminable (stack, views, references)
5. **Prioritize by impact** — High-frequency allocations first.
6. **Consider tradeoffs** — Clarity, maintainability, and dependencies.

## When NOT to Optimize

- **Architecture is still malleable** — Use **performance-design** before local
  tuning so representation choices do not cap the possible gain.
- **Cold paths** — Rarely-run code seldom deserves intricate local tuning.
- **No credible workload** — Preserve simple, favorable representations rather
  than adding speculative machinery.
- **Readability sacrifice** — Maintenance cost may exceed a marginal gain.
- **Dependency bloat** — Don't add crates for marginal gains.

## Escalation

- **Detailed profiling**: Consult **documentation-nerd**
- **Architectural performance**: Load **performance-design**
- **Testing optimized code**: Consult **testing-guru**
