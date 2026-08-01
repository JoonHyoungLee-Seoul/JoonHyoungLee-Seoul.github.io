---
layout: single
title: "[Compiler] 7.4 Loops and Strongly Connected Components"
date: 2026-05-31
categories: Compiler
tags: [Compiler, Control-Flow-Analysis]
---

> **Source note:** This article is a study note based on Section 7.4 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Section 7.4 explains how a compiler identifies loops in a control-flow graph using **back edges**, **natural loops**, and **strongly connected components**.

The key idea is that loops are not just “backward jumps.” A compiler needs a precise graph-theoretic definition so that later optimizations can safely reason about loop bodies, loop headers, loop nesting, and multiple-entry cycles.

This section mainly introduces two levels of loop structure:

```text
Natural Loop
    ↓
single-entry loop identified by a back edge

Strongly Connected Component
    ↓
general cyclic region, including multiple-entry loops
```

The section is based on Muchnick’s discussion of loops and SCCs in Chapter 7, Section 7.4.

---

## Back Edges and Natural Loops

A **back edge** is defined as an edge whose **head dominates its tail**.

In other words, for an edge:

```text
m → n
```

it is a back edge if:

```text
n dominates m
```

This definition is stricter than simply saying “an edge that points backward in DFS order.” A DFS back edge may suggest a cycle, but it does not always define a **natural loop**. If the loop has more than one entry point, it is not considered a natural loop.

### Natural Loop Definition

Given a back edge:

```text
m → n
```

the **natural loop** consists of:

- the loop header `n`,
    
- the node `m`,
    
- every node from which `m` can be reached without passing through `n`,
    
- and the edges connecting those nodes.
    

The node `n` is called the **loop header**.

Conceptually:

```text
        n   ← loop header
        ↓
      ...
        ↓
        m
        ↺
      back edge m → n
```

A natural loop is important because it has a **single entry point**: the header.

---

## Computing a Natural Loop

The algorithm `Nat_Loop(m, n, Pred)` computes the set of nodes in the natural loop for a back edge `m → n`.

The algorithm works backward from `m` through predecessor edges:

```text
Start with:
Loop = {m, n}

Then repeatedly:
    take a node p from the stack
    add predecessors of p
    continue until no new nodes are found
```

Because `n` dominates `m`, walking backward from `m` only collects nodes that are truly inside the loop. The algorithm therefore reconstructs the loop body from the back edge and predecessor relation.

### Intuition

Think of the back edge as marking the point where control returns to the loop header.

```text
Find back edge
    ↓
Start from tail of back edge
    ↓
Walk backward through predecessors
    ↓
Stop when all loop nodes are collected
```

This gives the compiler a concrete loop region that later optimizations can analyze.

---

## Preheaders

Many loop optimizations need to move code from inside a loop to a place **just before the loop starts**.

For example, loop-invariant code motion may transform:

```cpp
while (...) {
    x = a + b;   // same every iteration
    ...
}
```

into:

```cpp
x = a + b;
while (...) {
    ...
}
```

To make this transformation uniform, compilers introduce a **preheader**.

A **preheader** is a new basic block placed immediately before the loop header. All edges from outside the loop that previously entered the header are redirected to the preheader, and the preheader has a single edge to the loop header.

### Before

```text
outside blocks
      ↓
   header
      ↓
   loop body
      ↺
```

### After

```text
outside blocks
      ↓
   preheader
      ↓
   header
      ↓
   loop body
      ↺
```

The preheader gives the optimizer a safe and canonical location for hoisted computations.

---

## Loop Nesting and Same-Header Loops

Natural loops usually have a clean nesting relationship.

If two natural loops have **different headers**, then they are either:

```text
disjoint
```

or:

```text
one loop is nested inside the other
```

However, loops with the **same header** are ambiguous. The same flowgraph can come from source code where one loop is naturally nested inside the other, or from source code where the two cycles should be treated as one larger loop.

Because this distinction may not be recoverable from the flowgraph alone, this section treats same-header natural loops as a **single loop**. Structural analysis, introduced later in Section 7.7, handles such cases more precisely.

### Practical Meaning

For this section:

```text
same header + multiple back edges
        ↓
treat as one combined loop
```

This avoids making unjustified assumptions about the original source-level loop structure.

---

## Strongly Connected Components

A **natural loop** is only one kind of cyclic structure. Some loops have multiple entry points and therefore cannot be represented as natural loops.

To describe all possible cyclic regions, the section introduces **strongly connected components**, or **SCCs**.

A strongly connected component is a subgraph where every node can reach every other node using only edges inside that subgraph.

### Intuition

A set of blocks forms an SCC if control can circulate among all of them:

```text
B1 → B2
↑    ↓
B3 ←
```

From any block in the SCC, there is a path to every other block in the same SCC.

This is the most general way to describe loops in a flowgraph.

---

## Maximal SCCs

An SCC is **maximal** if it is not contained inside a larger SCC.

For example, a single self-loop block may technically be an SCC:

```text
B2 ↺
```

But if `B2` is part of a larger cyclic region involving `B1`, `B2`, and `B3`, then `{B2}` is not maximal.

The compiler is usually interested in **maximal SCCs**, because they represent the largest cyclic regions that should be treated as complete loop-like structures.

```text
Small SCC inside bigger SCC
        ↓
not maximal

Largest cyclic component
        ↓
maximal SCC
```

---

## Tarjan’s SCC Algorithm

The section presents `Strong_Components(r, Succ)`, a version of **Tarjan’s algorithm**, to compute all maximal SCCs in a flowgraph.

The algorithm uses:

- `Dfn(x)`: the depth-first preorder number of node `x`,
    
- `LowLink(x)`: the smallest preorder number reachable from `x` through DFS-tree paths plus at most one back or cross edge,
    
- `Stack`: nodes currently being considered for an SCC.
    

A node `x` is the root of an SCC when:

```text
LowLink(x) = Dfn(x)
```

At that point, nodes are popped from the stack to form one SCC.

### Algorithm Flow

```text
DFS visit node
    ↓
assign Dfn and LowLink
    ↓
visit successors
    ↓
update LowLink
    ↓
if LowLink(x) == Dfn(x):
        x is SCC root
        pop SCC from stack
```

The important property is efficiency: Tarjan’s algorithm computes SCCs in **linear time** with respect to the number of nodes and edges in the flowgraph.

---

## Why SCCs Matter for Compiler Optimization

Natural loops are enough for many common loop optimizations, but they do not cover all possible control-flow cycles.

SCCs matter because they allow the compiler to reason about:

- multiple-entry loops,
    
- irreducible control flow,
    
- general cyclic regions,
    
- loop-like regions not expressible as natural loops.
    

This becomes important in the next section on **reducibility**. A reducible flowgraph is one where all loops are natural loops characterized by back edges. If a cyclic region is an SCC but not a natural loop, the graph may be irreducible.

Conceptually:

```text
Natural loops
    ↓
well-structured loops

SCCs
    ↓
all cyclic regions, including irregular ones
```

---

## Key Takeaways

### 1. A Back Edge Requires Dominance

A back edge is not merely a DFS backward edge. In this section, an edge `m → n` is a back edge only when `n` dominates `m`.

### 2. Natural Loops Are Single-Entry Loops

A natural loop is built from a back edge and has a unique entry point: the loop header.

### 3. Preheaders Make Loop Optimizations Cleaner

A preheader provides a canonical place to move code before the loop, which is essential for transformations such as loop-invariant code motion.

### 4. Same-Header Loops Are Ambiguous

When multiple natural loops share the same header, this section treats them as a single loop because the flowgraph alone may not reveal the original source-level nesting.

### 5. SCCs Generalize Loops

Strongly connected components describe all cyclic regions in a flowgraph, including loops that are not natural loops.

### 6. Tarjan’s Algorithm Computes SCCs Efficiently

The section uses a Tarjan-style DFS algorithm with `Dfn`, `LowLink`, and a stack to compute maximal SCCs in linear time.
