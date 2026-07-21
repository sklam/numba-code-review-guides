# Pointer Arithmetic and Provenance Guide for Numba Code Reviews

**Document Purpose**: Educational reference for code reviewers and contributors working with LLVM pointer operations in Numba, based on lessons from PR #10696 and issues #10695, #10605.

**Triggers**: `ptrtoint` · `inttoptr` · `pointer_add` · `builder.add(` / `builder.sub(` on a pointer ·
pointer stored in an `intp_t` `alloca` · `bitcast` for cast-plus-offset.

**Key References**:
- GitHub PR #10696: "Use gep in pointer_add"
- GitHub Issue #10695: "MemoryError from 'A'-layout array access"
- GitHub Issue #10605: "Nested function with sum on expand_dims hangs or gives wrong result"

---

## The Problem: Pointer Provenance

**Pointer provenance** is LLVM's way of tracking which object a pointer came from. The optimizer relies on it for alias analysis, vectorization/DCE decisions, and memory-safety reasoning. When provenance is lost, LLVM can miscompile: delete "impossible" branches, fabricate spurious `MemoryError`s, or eliminate whole compute loops.

### The Critical Rule

**Never convert pointers to integers and back to do pointer arithmetic.** Use a GEP-based helper so provenance stays attached to the pointer.

```python
# ❌ WRONG — provenance goes through an integer
intptr = builder.ptrtoint(ptr, intp_t)
intptr = builder.add(intptr, offset)
result = builder.inttoptr(intptr, ptr.type)

# ✅ CORRECT — provenance stays on the pointer
result = cgutils.pointer_add(builder, ptr, offset)
```

### Why `ptrtoint`/`inttoptr` misbehaves

At the IR level the two forms diverge:

```llvm
; ptrtoint/inttoptr — provenance routed through an integer:
%int     = ptrtoint i8* %ptr to i64
%new_int = add i64 %int, %offset
%new_ptr = inttoptr i64 %new_int to i8*
; In theory inttoptr should recover %ptr's provenance, but LLVM currently
; ignores the "exposure" side-effect of ptrtoint. Passes like SCEVExpander
; then rematerialize the address as a GEP off a *null* base, which folds to
; `poison` and lets later passes collapse the surrounding branch
; (e.g. the null-check after an allocation).  →  miscompilation

; GEP — provenance stays attached:
%new_ptr = getelementptr i8, i8* %ptr, i64 %offset
; result is derived directly from %ptr; no integer round-trip.  →  correct
```

**The semantics are not fully settled** — a caution, not a fixed rule. The `inttoptr(ptrtoint(p) + offset)` form is *arguably* legal (the offset carries `p`'s provenance), but LLVM's provenance model is itself under active development. Upstream work in 2025 is tightening, not relaxing, this area:

- **`ptrtoaddr`** — a side-effect-free counterpart to `ptrtoint` that returns only the address without exposing provenance (cf. Rust's `addr()` vs `expose_provenance()`).
- **The `ptradd` migration** — GEPs canonicalizing toward a single byte offset (`getelementptr i8, ptr %p, i64 OFF`), with a dedicated `ptradd` instruction planned. Our GEP-based `pointer_add()` already aligns with this.
- **A formal provenance-model working group** formed late in 2025; a draft model exists and work is ongoing.

Because the rules are being *firmed up*, arithmetic that merely *appears* to work today may become exposed to miscompilation. Preferring GEP keeps provenance explicit and on the safe side of the change. See the [Numba-side analysis](https://github.com/numba/numba/issues/10695#issuecomment-4984483266) and the [upstream status](https://www.npopov.com/2026/01/31/This-year-in-LLVM-2025.html#ptrtoaddr).

---

## What Went Wrong

Before PR #10696, `pointer_add()` computed the offset via `ptrtoint` → `add` → `inttoptr`, losing provenance. Two field failures resulted:

**Issue #10695 — `MemoryError` on a valid allocation.** A tiny `(2,2,2)` array raised `MemoryError: Allocation failed (probably too large)`. Non-contiguous ('A'-layout) access used `pointer_add`; inlining exposed the lost provenance, LLVM miscompiled the post-allocation null-check so the success branch jumped straight to the error handler, and the compute loop was deleted. Triggered only under nested `njit` (aggressive inlining) + 'A' layout + constant folding, and only on x86-64 — a pass-order-specific bug.

**Issue #10605 — nested function hangs / wrong result.** A function correct standalone miscompiled when called from another jitted function:
```python
@numba.njit
def f(x):
    return np.expand_dims(x, 0).sum()

@numba.njit
def g(x):
    return f(x)          # inlining exposes provenance-less pointer arithmetic

x = np.array([[1.0, 0.0], [3.0, 4.0]])
f(x)   # ✅ 8.0
g(x)   # ❌ hangs or returns 1.0
```

Same root cause: inlining exposed the arithmetic, loop optimizations made wrong assumptions, and the compute loop was eliminated or miscompiled.

---

## The Solution

PR #10696 reimplemented `pointer_add()` to bitcast the pointer to `i8*`, add the offset with a **byte-wise GEP**, then bitcast back to the requested type. GEP preserves provenance while offsetting.

### Important constraint

**GEP is only valid for "in-object" offsets** — the result must stay within the same allocated object `ptr` came from. Fine for array-element access, struct-field access, and strided iteration within an array. **Not** valid for arbitrary arithmetic across objects or for implementing allocator internals. All existing Numba uses are in-object, as documented in PR #10696.

---

## Common Patterns

### Array element access
```python
# ❌  data_int = ptrtoint(array.data); elem = inttoptr(add(data_int, mul(index, itemsize)))
offset   = builder.mul(index, stride)
elem_ptr = cgutils.pointer_add(builder, array.data, offset)     # ✅
# better still, the high-level helper:
elem_ptr = cgutils.get_item_pointer2(context, builder, data, shape, strides, layout, indices)
```

### Strided iteration
```python
# ❌  store the pointer as an integer, advance with add(), read back with inttoptr
# ✅  store the pointer as a pointer, advance with pointer_add
class ArrayIterator:
    def __init__(self, ...):
        self.iter_ptr = cgutils.alloca_once(builder, ary.data.type)   # pointer-typed alloca
        builder.store(ary.data, self.iter_ptr)

    def advance(self):
        ptr = builder.load(self.iter_ptr)
        builder.store(cgutils.pointer_add(builder, ptr, stride), self.iter_ptr)

    def get_current(self):
        return builder.load(self.iter_ptr)   # direct load, no inttoptr
```

### Negative offsets
```python
# ❌  builder.sub(ptr_as_int, offset)          # integer subtraction on a pointer
ptr        = builder.load(iter_ptr)
offset     = builder.mul(stride, count)
neg_offset = builder.sub(Constant(intp_t, 0), offset)         # negate, then...
new_ptr    = cgutils.pointer_add(builder, ptr, neg_offset)    # ✅
```

### Type cast (± offset)
```python
# ❌  inttoptr(ptrtoint(ptr), new_type)
new_ptr = builder.bitcast(ptr, new_type)                          # ✅ cast only
new_ptr = cgutils.pointer_add(builder, ptr, offset, return_type=new_type)  # ✅ cast + offset
```

---

## Review Checklist

### Storage
- [ ] Pointers stored as pointer types, not integers (`alloca` uses the pointer type, not `intp_t`)

### Arithmetic
- [ ] No `ptrtoint` before / `inttoptr` after pointer arithmetic
- [ ] Offsets computed via `cgutils.pointer_add()`
- [ ] Negative offsets via `builder.sub(0, offset)`, not integer subtraction on the pointer

### Access & casts
- [ ] Loads/stores go through pointer-typed `alloca`s (no `inttoptr`/`ptrtoint` round-trip)
- [ ] Type changes use `builder.bitcast()`; cast-plus-offset uses `pointer_add(..., return_type=)`
- [ ] All offsets are in-object (within the allocation `ptr` came from)

---

## Testing Notes

Direct unit tests for provenance preservation are impractical: the bugs need specific LLVM pass sequences and loop-nest structures, so any reproducer is fragile. As PR #10696 notes, relying on the existing suite is fine. Do exercise functional correctness across platforms (x86-64 **and** ARM64 — these bugs are platform-specific) and across LLVM versions; don't expect a unit test to catch future optimizer changes.

---

## Quick Reference

| Operation | ❌ Wrong | ✅ Right |
|-----------|---------|---------|
| Offset pointer | `inttoptr(add(ptrtoint(ptr), offset))` | `cgutils.pointer_add(builder, ptr, offset)` |
| Store pointer | `store(ptrtoint(ptr), intp_alloca)` | `store(ptr, ptr_alloca)` |
| Load pointer | `inttoptr(load(intp_alloca))` | `load(ptr_alloca)` |
| Advance iterator | `store(add(load(iter_int), stride), iter_int)` | `store(pointer_add(load(iter_ptr), stride), iter_ptr)` |
| Type cast | `inttoptr(ptrtoint(ptr), new_type)` | `bitcast(ptr, new_type)` |
| Negative offset | `sub(ptr_as_int, offset)` | `pointer_add(ptr, sub(0, offset))` |

---

## Further Reading

- **PR #10696**: https://github.com/numba/numba/pull/10696
- **Issue #10695**: https://github.com/numba/numba/issues/10695
- **Issue #10605**: https://github.com/numba/numba/issues/10605
- **LLVM Language Reference (GEP)**: https://llvm.org/docs/LangRef.html#getelementptr-instruction
- **LLVM Pointer Aliasing**: https://llvm.org/docs/AliasAnalysis.html
- **"This year in LLVM (2025)" — provenance, `ptrtoaddr`, `ptradd`**: https://www.npopov.com/2026/01/31/This-year-in-LLVM-2025.html#ptrtoaddr

---

## Conclusion

**Golden Rules**
1. **Preserve provenance** — GEP-based `pointer_add()`, never `ptrtoint`/`inttoptr` arithmetic.
2. **Store pointers as pointers**, not integers.
3. **Stay ahead of the optimizer** — LLVM's provenance model is tightening; keep arithmetic provenance-preserving so future changes can't miscompile it.
4. **Test broadly** — one platform passing doesn't mean it's correct.

> "The old `pointer_add()` might just be relying on an undefined behavior" — @sklam, PR #10696

Whether the old form is *strictly* illegal is arguable; that's exactly why GEP is the safe choice as the provenance model firms up upstream.
