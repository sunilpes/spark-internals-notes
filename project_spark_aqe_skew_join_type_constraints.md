---
name: AQE Skew Join Type Constraints
description: Why AQE skew optimization doesn't support all join types — correctness constraints around NULL generation, partial data producing wrong NULLs and duplicates for outer joins, with concrete examples
type: project
---

## Why AQE Skew Optimization Doesn't Support All Join Types

### The Core Issue: NULL Correctness

Splitting a partition means one task only sees **partial data** for that key range. For joins that emit NULLs for non-matching rows, partial data produces **wrong results**.

### Inner Join — Can Split Both Sides

Only outputs matches. Partial data finds partial matches, union is correct.

```
L2-1 ⋈ R2 (full) → matches for L2-1 ✓
L2-2 ⋈ R2 (full) → matches for L2-2 ✓
Union → same as L2 ⋈ R2 ✓
```

### LeftOuter — Can Only Split LEFT

**Split LEFT (safe):** Each left row appears exactly once, gets NULLs only if truly no match in full R2.

**Split RIGHT (WRONG):**
```
L2 (full) ⋈ R2-1 (partial) → left row gets NULL (no match in R2-1)
L2 (full) ⋈ R2-2 (partial) → left row matches R2-2

Union → duplicate left rows, one with wrong NULL ✗
```

### FullOuter — Can't Split Either Side

Both sides need to see the full other side for correct NULL generation.

```
Splitting LEFT:
  L2-1 ⋈ R2 → right rows not matching L2-1 get NULLs
  L2-2 ⋈ R2 → right rows not matching L2-2 get NULLs
  But right row might match L2-2 yet got NULL from L2-1 → WRONG
```

### The Pattern

```
Can split side X if:
  - Each X row's output depends only on itself + full other side
  - Other side doesn't need to see ALL of X to determine NULLs

Cannot split side X if:
  - The OTHER side needs ALL of X to decide whether to emit NULLs
```

### Summary Table

| Join Type | Split Left? | Split Right? | Why |
|-----------|------------|-------------|-----|
| Inner | Yes | Yes | Only matches — partial is fine |
| Cross | Yes | Yes | No condition — partial is fine |
| LeftOuter | Yes | No | Splitting right → false NULLs for left rows |
| RightOuter | No | Yes | Mirror of LeftOuter |
| LeftSemi | Yes | No | Need full right to confirm "any match exists" |
| LeftAnti | Yes | No | Need full right to confirm "no match exists" |
| FullOuter | No | No | Both sides need full other for correct NULLs |
