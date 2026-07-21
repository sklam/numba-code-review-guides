# Heap Reads/Writes Must Convert Between Data and Value Repr

**Triggers**: `builder.load(` · `builder.store(` on a heap pointer · `load_item` ·
`store_item` · `from_data(` · `as_data(` · a boolean dtype in codegen.

**Purpose**: A focused rule extending [review-data-model.md](review-data-model.md) —
especially *Mistake #3 (raw `builder.store` on array pointers)* and
*Pattern 3 (load_item/store_item)*. This is the **load** side of that mistake.

---

## The Rule

> A pointer into **heap memory** holds the **data repr**. Computation needs the **value
> repr**. Every heap read must convert data→value on the way in; every heap write must
> convert value→data on the way out. Raw llvmlite `builder.load` / `builder.store`
> (`IRBuilder`) move bytes with **no conversion**, so they are wrong on any heap pointer.

Use the right converter for the kind of heap data:

| Heap data | Read | Write |
|---|---|---|
| Array element buffer | `load_item(ctx, bld, arrayty, ptr)` | `store_item(ctx, bld, arrayty, val, ptr)` |
| Any other heap value | `datamodel.from_data(bld, builder.load(ptr))` | `builder.store(datamodel.as_data(bld, val), ptr)` |

`load_item`/`store_item` are just the array-typed wrappers: they load/store the raw bytes
and apply `from_data`/`as_data` for you. For non-array heap data (struct members, boxed
values, NRT-managed payloads) call `from_data`/`as_data` directly around the load/store.

Stack `alloca`s and SSA values are *already* value repr — those use plain
`builder.load`/`builder.store` with no conversion (see the base guide).

## Why It Hides

For most dtypes the value repr and data repr are the *same* LLVM type (e.g. `float64` is
`double`, `int64` is `i64`), so a raw load happens to produce the right type and the bug
is masked. It only surfaces when the two reprs differ — most commonly **boolean**:
`i1` in computation (value repr) but `i8` in the heap buffer (data repr).

Symptoms of the mismatch:
- Caught at IR-build time when the wrong-typed value feeds a binary op — llvmlite raises
  `"Operands must be the same type, got (i8, i1)"`.
- If it instead feeds a `store`, IR may be emitted silently wrong — **no crash is not
  proof of correctness.**

## Corollaries

- Read and write of the same buffer must be a **matched pair**: `from_data` on read
  pairs with `as_data` on write (`load_item` with `store_item`). A raw `builder.load`
  after a converting store reads the wrong repr.
- Use the data model that owns the value being loaded/stored (for arrays, the `arrayty`
  that owns the buffer).
- "It works on int/float" proves nothing about repr correctness. Always exercise a
  **boolean** dtype in codegen that reads/writes heap data.
- Select dtype branches with a direct type check —
  `isinstance(dtype, types.Boolean)` — not `data_model_manager[...]._fe_type` (that is
  *Mistake #1* from the base guide).

---

## Review Checklist

Extends the *Array Operations* section of [review-data-model.md](review-data-model.md):

- [ ] Every read from a heap pointer converts data→value; every write converts
      value→data — no bare `builder.load`/`builder.store` on heap pointers.
- [ ] Array buffers use `load_item`/`store_item`; other heap data uses
      `from_data`/`as_data` around the load/store.
- [ ] Read/write of the same buffer use a **matched** converter pair.
- [ ] The data model / `arrayty` used matches the value that owns the buffer.
- [ ] Operands of binary ops (`or_`, `add`, …) are all in **value repr** — none is a raw
      heap load.
- [ ] A **boolean** dtype is exercised by tests to catch `i1`/`i8` mismatches.
- [ ] dtype branch selection uses `isinstance(dtype, types.Boolean)`.

---

**Extends**: review-data-model.md
