---
layout: single
title: "[Compiler] 12.5 Copy Propagation"
date: 2026-05-28 20:33:33 +0800
categories: Compiler
tags: [Compiler, Optimization]
toc: true
toc_sticky: true
---

## Core Idea

**Copy propagation** replaces later uses of a variable with the variable from which its value was copied.

Given a copy assignment:

```text
x ← y
```

later uses of `x` may be rewritten as uses of `y`, provided that neither `x` nor `y` has been assigned a new value along the relevant path.

### Example

#### Before Copy Propagation

```text
b ← a
c ← b + 1
d ← b
```

#### After Copy Propagation

```text
b ← a
c ← a + 1
d ← a
```

The assignment `b ← a` may become unnecessary after propagation and can later be removed by dead-code elimination.

Copy propagation does not compute a new value. Instead, it exposes the original producer of a value directly to later instructions, simplifying the intermediate representation and enabling later optimizations. In the textbook, Figure 12.23 illustrates this transformation by propagating the copy `b ← a` from one block into later uses.

---

## Relationship to Register Coalescing

Copy propagation is closely related to **register coalescing**.

When optimization operates on low-level intermediate code in which variables are already represented as symbolic or physical registers, the two transformations can have the same observable effect:

```text
r2 ← r1
use r2
```

may become:

```text
r2 ← r1
use r1
```

However, their applicability is determined differently:

- **Copy propagation** uses data-flow information to determine whether a copy is valid at a later use.
    
- **Register coalescing** uses an interference graph to determine whether the source and destination can safely share one register.
    

Thus, copy propagation is fundamentally a value-flow transformation, while register coalescing is primarily a register-allocation transformation.

---

## Local Copy Propagation

Local copy propagation operates within a single basic block. Because control does not branch inside the block, the compiler can scan instructions in order while maintaining the set of currently valid copies.

The textbook represents this set as `ACP`, the set of **available copies**:

```text
ACP = { ⟨destination, source⟩ }
```

For example:

```text
ACP = { ⟨b, a⟩, ⟨d, a⟩ }
```

means that current uses of `b` and `d` may be replaced with `a`.

### Processing Each Instruction

For each instruction in a basic block, the local algorithm performs three steps:

```text
1. Replace source operands using currently available copies.
2. Remove copies invalidated by the instruction's assignment.
3. If the instruction is a valid copy assignment, add it to ACP.
```

A copy `⟨x, y⟩` becomes invalid if a later instruction assigns to either `x` or `y`.

```text
x ← y        ACP includes ⟨x, y⟩
...
x ← z        remove ⟨x, y⟩ because x changed
```

or:

```text
x ← y        ACP includes ⟨x, y⟩
...
y ← z        remove ⟨x, y⟩ because the copied value changed
```

### Linear-Time Local Algorithm

Figure 12.24 presents an `O(n)` local copy-propagation algorithm for a block of `n` instructions. Its main components are:

- `Local_Copy_Prop`, which scans the block;
    
- `Copy_Value`, which substitutes a variable when a matching available copy exists;
    
- `Remove_ACP`, which removes invalidated copies.
    

The essential local workflow is:

```text
Scan instruction
    ↓
Propagate available copies into operands
    ↓
Kill invalid copies if a variable is redefined
    ↓
Record a newly discovered copy assignment
```

---

## Local Example

Figure 12.25 demonstrates how available copies evolve while scanning one basic block.

### Before Optimization

```text
1  b ← a
2  c ← b + 1
3  d ← b
4  b ← d + c
5  b ← d
```

### Propagation Process

|Position|Available Copies Before Instruction|Optimized Instruction|
|--:|---|---|
|1|`∅`|`b ← a`|
|2|`{⟨b, a⟩}`|`c ← a + 1`|
|3|`{⟨b, a⟩}`|`d ← a`|
|4|`{⟨b, a⟩, ⟨d, a⟩}`|`b ← a + c`|
|5|`{⟨d, a⟩}`|`b ← a`|

### Optimized Block

```text
1  b ← a
2  c ← a + 1
3  d ← a
4  b ← a + c
5  b ← a
```

At instruction 4, the new assignment to `b` invalidates the previously available copy `⟨b, a⟩`, but the copy `⟨d, a⟩` remains valid. Therefore instruction 5 can still replace `d` with `a`.

---

## Global Copy Propagation

Local propagation cannot propagate copies across basic-block boundaries. **Global copy propagation** extends the transformation across the control-flow graph by determining which copy assignments remain valid at the entry and exit of every block.

A copy assignment may be propagated into a block only when it reaches that block along **every incoming path** without either variable being redefined.

### Required Information

For each basic block `i`, the algorithm computes two sets:

- `COPY(i)`: copy assignments generated in block `i` that remain valid at the end of the block.
    
- `KILL(i)`: copy assignments generated elsewhere that are invalidated by assignments in block `i`.
    

A copy instance is represented as a quadruple:

```text
⟨destination, source, block, position⟩
```

For example:

```text
⟨d, c, B1, 2⟩
```

means that instruction 2 of block `B1` contains the copy assignment:

```text
d ← c
```

and that the assignment reaches the end of `B1` without either `d` or `c` being redefined.

---

## Data-Flow Equations

The analysis computes:

- `CPin(i)`: copies available when entering block `i`;
    
- `CPout(i)`: copies available when leaving block `i`.
    

Because a copy is safe to propagate into a block only if it is valid along **all** predecessor paths, the meet operator is set intersection.

[  
CP_{in}(i) = \bigcap_{j \in Pred(i)} CP_{out}(j)  
]

[  
CP_{out}(i) = COPY(i) \cup \left(CP_{in}(i) - KILL(i)\right)  
]

### Initialization

```text
CPin(entry) = ∅
CPin(i)     = U, for every block i ≠ entry
```

where `U` is the universal set of copy-assignment instances, or at least the union of all `COPY(i)` sets.

### Interpretation

```text
A copy reaches a block from every predecessor
            ↓
The block does not redefine either copied variable
            ↓
The copy remains available for propagation
```

Because the analysis manipulates finite sets of copy instances, it can be implemented efficiently using bit vectors.

---

## Applying Global Propagation

Once `CPin` has been computed, the compiler reuses the local propagation machinery.

For each basic block `B`:

```text
1. Initialize ACP using the copies represented in CPin(B).
2. Run the local copy-propagation scan through B.
```

The only difference from purely local propagation is that `ACP` is no longer initialized to the empty set. It begins with copies already known to be valid at the block entry.

### Example from the Textbook

In Figure 12.26, the analysis derives entry information such as:

```text
CPin(B2) = { ⟨d, c, B1, 2⟩ }

CPin(B3) = {
    ⟨d, c, B1, 2⟩,
    ⟨g, e, B2, 2⟩
}
```

This means that uses of `d` in `B2` may be replaced by `c`, and uses of both `d` and `g` in `B3` may be replaced by `c` and `e`, respectively.

Figure 12.27 shows the resulting control-flow graph after local propagation in `B1` and global propagation across the procedure.

---

## Extension to Extended Basic Blocks

The local algorithm can be generalized from individual basic blocks to **extended basic blocks**.

An extended basic block is a region with one entry but possibly multiple exits. Because every non-entry block in the region has only one predecessor within the region, available-copy information can be propagated directly during a preorder traversal.

### Local Processing over an Extended Basic Block

```text
Process the entry block
    ↓
Pass its final ACP state to its successor blocks
    ↓
Continue in preorder through the region
```

This shifts more propagation work from the global data-flow phase into the cheaper local phase.

For global propagation over extended basic blocks, the analysis must maintain a separate `CPout` set for each exit, because different paths through the region may make different copies available.

In the textbook example, blocks `B2`, `B3`, `B4`, and `B6` form an extended basic block. Processing them locally allows the copy assigning `e` to `g` in `B2` to be propagated throughout that region.

---

## Limitation and Phase-Ordering Issue

Global copy propagation does not detect every semantically equivalent copy situation.

Figure 12.28 presents a case in which two branches independently execute the same copy:

```text
B2: x ← y
B3: x ← y
```

Although `x` has the value of `y` regardless of which branch is taken, neither individual copy assignment is available on all incoming paths to the later block. Therefore ordinary global copy propagation cannot propagate `y` into that block.

### Transformations That Expose the Opportunity

The missed opportunity can be recovered by moving or merging the duplicate assignments:

```text
Tail Merging
    ↓
One shared copy assignment
    ↓
Copy Propagation can recognize it
```

Alternatively:

```text
Partial-Redundancy Elimination or Code Hoisting
    ↓
Move x ← y before the branch
    ↓
Copy Propagation can use it globally
```

This creates a **phase-ordering problem**: tail merging is commonly performed only after machine instruction generation, whereas copy propagation is generally performed earlier. Partial-redundancy elimination or code hoisting can expose the same opportunity within the earlier optimization phase.

---

## Key Takeaways

### 1. Copy Propagation Substitutes Names, Not Computations

Given `x ← y`, later uses of `x` can become uses of `y` while the copy remains valid.

### 2. Local Propagation Is a Forward Scan

Within one basic block, the compiler maintains an available-copy set `ACP`, substitutes operands, removes invalidated copies, and records new ones.

### 3. Global Propagation Requires Must-Availability

Across blocks, a copy must reach a use unchanged along every incoming path. This is why the global data-flow meet operator is intersection.

### 4. Existing Local Logic Can Be Reused Globally

Once `CPin` is computed, global copy propagation initializes each block's `ACP` with its available incoming copies and applies the local algorithm.

### 5. Other Optimizations May Be Needed First

Equivalent copies occurring on separate control-flow paths may not be detected until tail merging, partial-redundancy elimination, or code hoisting transforms the control-flow graph.