# Data Model Usage Guide for Numba Code Reviews

**Document Purpose**: Educational reference for code reviewers and contributors working with Numba's data model system, based on lessons from PR #10657 review.

**Key References**:
- GitHub PR #10657, Review #4715817475 by @sklam

---

## Core Concepts

The **data model** (`context.data_model_manager`) defines how Numba types are represented at the LLVM IR level. **Use it only when deciding LLVM-level types — never for frontend type operations.**

The distinction that drives every rule below is **value** vs **data** representation:

| | Value repr | Data repr |
|---|---|---|
| What | SSA values, local stack `alloca`s | Heap memory (malloc'd) |
| Where | temporaries in the current computation | array data, values passed between functions or back to Python |
| Lifetime | within one function | can outlive the function; managed by NRT |
| Load / store | `builder.load()` / `builder.store()` | `load_item()` / `store_item()` (→ `context.unpack_value` / `pack_value`) |

> "`value` repr is for SSA-value and local stack allocations (`alloca`s) — temporaries for the current computation. `data` is for data stored in heap memory — exposed to other functions or back to Python, so it needs a consistent representation." — @sklam, PR #10657

The conversions `from_data()` / `as_data()` translate between the two. Local `alloca`s and SSA values are *already* value repr, so they need no conversion.

---

## Common Mistakes

### ❌ Using the data model for type checking
```python
if context.data_model_manager[ret_dtype]._fe_type is types.boolean:  # ❌ needless indirection
    ...
```
`._fe_type` just returns the original frontend type. Check it directly:
```python
if isinstance(ret_dtype, types.Boolean):   # ✅  (or: ret_dtype == types.boolean)
    ...
```

### ❌ Converting around a stack `alloca`
```python
result = cgutils.alloca_once_value(builder, zero)   # value repr already
res_val = context.data_model_manager[ret_dtype].from_data(builder, builder.load(result))  # ❌
...
data_val = context.data_model_manager[ret_dtype].as_data(builder, value)  # ❌
builder.store(data_val, result)
```
An `alloca` holds value repr. `from_data()`/`as_data()` expect/produce *data* repr for heap — using them here is both unnecessary and a type mismatch. Load and store directly:
```python
res_val = builder.load(result)     # ✅ already value repr
builder.store(value, result)       # ✅ direct store of value
```

### ❌ Raw `builder.store()` into array (heap) data
```python
ptr = cgutils.get_item_pointer2(...)   # pointer into array
builder.store(value, ptr)              # ❌ skips data-model conversion + alignment
```
Array data is heap memory needing data repr. Use the paired helper, which calls `context.pack_value()` and respects alignment:
```python
store_item(context, builder, arrayty, value, ptr)   # ✅ pairs with load_item()
```

---

## Correct Pattern: Mixing Stack and Heap

The only idiom worth showing whole — everything else is a direct load/store:

```python
def codegen(context, builder, sig, args):
    ary = args[0]
    zero = context.get_constant(ret_dtype, 0)
    accumulator = cgutils.alloca_once_value(builder, zero)   # stack
    res = _empty_nd_impl(context, builder, result_type, shape)  # heap

    with ArrayIterator(context, builder, aryty, ary, (res,)) as (in_ptr, (out_ptr,)):
        val = load_item(context, builder, aryty, in_ptr)   # heap → value repr
        acc_val = builder.load(accumulator)                # stack → already value repr
        new_acc = builder.add(acc_val, val)                # arithmetic on value repr
        builder.store(new_acc, accumulator)                # → stack, no conversion
        store_item(context, builder, result_type, new_acc, out_ptr)  # → heap, converts
```

---

## Review Checklist

### Type checking
- [ ] No `context.data_model_manager[type]._fe_type` for type checks — use `isinstance(...)` / `==`
- [ ] Data model accessed only when determining LLVM types

### Local variables (`alloca`) — value repr
- [ ] No `from_data()` wrapping a `builder.load()` from an `alloca`
- [ ] No `as_data()` before a `builder.store()` to an `alloca`
- [ ] Direct `builder.load()` / `builder.store()` for stack operations

### Array (heap) access — data repr
- [ ] `load_item()` when reading from array pointers, `store_item()` when writing
- [ ] Consistent pairing: `load_item()` implies `store_item()` on the write side

### Mixed
- [ ] Stack ↔ stack: no conversions
- [ ] Heap → stack: `load_item()` then direct `builder.store()`
- [ ] Stack → heap: direct `builder.load()` then `store_item()`
- [ ] `from_data`/`as_data` appear only inside data-model definitions or custom-type LLVM layouts — not in ordinary codegen

---

## Worked Example

From the PR #10657 review:

```python
# ❌ Incorrect
if context.data_model_manager[ret_dtype]._fe_type is types.boolean:  # data model for type check
    add_funcfn = lambda builder, args: builder.or_(*args)
else:
    add_funcfn = context.get_function(fnty, fn_sig)

with ArrayIterator(context, builder, aryty, ary) as iter_val_ptr:
    val = load_item(context, builder, aryty, iter_val_ptr)
    res_val = add_funcfn(builder, (
        context.data_model_manager[ret_dtype].from_data(       # from_data on an alloca
            builder, builder.load(result)),
        context.data_model_manager[ret_dtype].from_data(       # from_data on a value
            builder, context.cast(...))))
    builder.store(res_val, result)
```
```python
# ✅ Correct
if isinstance(ret_dtype, types.Boolean):                       # direct frontend type check
    add_funcfn = lambda builder, args: builder.or_(*args)
else:
    add_funcfn = context.get_function(fnty, fn_sig)

with ArrayIterator(context, builder, aryty, ary) as iter_val_ptr:
    val = load_item(context, builder, aryty, iter_val_ptr)
    res_val = add_funcfn(builder, (
        builder.load(result),   # direct load from stack (already value repr)
        context.cast(...)))     # cast returns value repr
    builder.store(res_val, result)
```

---

## Quick Reference

| Operation | Source → Target | Correct approach |
|-----------|-----------------|------------------|
| Type check | frontend type → decision | `isinstance(type, types.X)` |
| Alloca → compute | stack → SSA value | `builder.load(ptr)` |
| Compute → alloca | SSA value → stack | `builder.store(val, ptr)` |
| Array → compute | heap → SSA value | `load_item(ctx, bld, ty, ptr)` |
| Compute → array | SSA value → heap | `store_item(ctx, bld, ty, val, ptr)` |
| Alloca → array | stack → heap | `load` then `store_item` |
| Array → alloca | heap → stack | `load_item` then `store` |

---

## Further Reading

- `numba/core/datamodel/models.py`: data model implementations
- `numba/np/arrayobj.py`: `load_item()` / `store_item()` definitions
- `numba/core/pythonapi.py`: Python-object interfacing with data models
- `numba/core/cgutils.py`: `alloca_once_value()` and friends

---

## Conclusion

**Golden Rules**
1. Data models decide **LLVM types**, not frontend type checks.
2. `alloca`s and SSA values are **already value repr** — no conversions.
3. Always use **`load_item`/`store_item`** for array (heap) access.
4. Never mix `from_data`/`as_data` with stack operations.
