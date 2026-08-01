---
layout: single
title: "[Compiler] 7.3 Dominators and Postdominators Summary"
date: 2026-06-10 14:18:16 +0800
categories: Compiler
tags: [Compiler, Control-Flow-Analysis]
toc: true
toc_sticky: true
---



---
## Core Idea

Section 7.3 introduces **dominance**, one of the most important control-flow relations used by optimizing compilers.

A node `d` **dominates** a node `i`, written:

```text
d dom i
```

if **every possible execution path from `entry` to `i` must pass through `d`**. In other words, before the program can reach `i`, it must already have gone through `d`. This makes dominance useful for identifying loops, safe code motion points, and structural relationships inside a control-flow graph.

Dominance has three mathematical properties:

```text
reflexive:      every node dominates itself
transitive:     if a dom b and b dom c, then a dom c
antisymmetric:  if a dom b and b dom a, then a = b
```

Because of these properties, dominance relations can be represented as a tree rooted at `entry`.

---

## Dominators, Immediate Dominators, and the Dominator Tree

A node may have many dominators. For example, if every path to `B4` goes through:

```text
entry → B1 → B3 → B4
```

then the dominators of `B4` include:

```text
{entry, B1, B3, B4}
```

However, compiler algorithms usually need the **closest strict dominator** of each node. This is called the **immediate dominator**, written:

```text
idom(n)
```

The immediate dominator of a node `n` is the nearest node that must be passed through before reaching `n`.

```text
entry
  ↓
 B1
  ↓
 B3
  ↓
 B4
```

In this chain:

```text
idom(B4) = B3
idom(B3) = B1
idom(B1) = entry
```

The section defines immediate dominance formally and notes that the immediate dominator of a node is unique. The immediate dominance relation forms the **dominator tree**, where paths in the tree reveal dominance relationships.

The section also defines **strict dominance**:

```text
d sdom i
```

meaning:

```text
d dominates i, and d ≠ i
```

So every node dominates itself, but no node strictly dominates itself.

---

## Postdominators

A **postdominator** is the reverse-direction counterpart of a dominator.

A node `p` **postdominates** node `i`, written:

```text
p pdom i
```

if every possible execution path from `i` to `exit` passes through `p`.

So dominance asks:

```text
Must we pass through d before reaching i?
```

Postdominance asks:

```text
Must we pass through p after leaving i before exiting?
```

The textbook explains that postdominance can be computed by reversing all edges in the flowgraph and swapping `entry` and `exit`.

This becomes important later for **control dependence** and **program-dependence graphs**, because whether a node executes often depends on which predicates control paths that eventually reach or avoid postdominators.

---

## Simple Dominator Computation

The first algorithm in this section computes the full set of dominators for every node.

The basic idea is iterative:

```text
Domin(entry) = {entry}

For every other node n:
    Domin(n) initially contains all nodes

Repeatedly update:
    Domin(n) = {n} ∪ intersection of Domin(p) for all predecessors p of n
```

Conceptually:

```text
A node dominates n
only if it dominates every predecessor path into n.
```

So if a block has multiple incoming edges, a dominator must appear on **all** incoming paths.

### Algorithm Shape

```text
initialize dominator sets
repeat
    for each node n except entry
        compute intersection of dominators of predecessors
        add n itself
        update Domin(n)
until no set changes
```

The textbook notes that processing nodes in depth-first order improves efficiency. It also gives a concrete example using the flowgraph from Figure 7.4, producing dominator sets such as:

```text
Domin(entry) = {entry}
Domin(B1)    = {entry, B1}
Domin(B2)    = {entry, B1, B2}
Domin(B3)    = {entry, B1, B3}
Domin(B4)    = {entry, B1, B3, B4}
```

This shows the central intuition: each node’s dominator set records all blocks that execution must pass before reaching that node.

---

## Computing Immediate Dominators from Dominator Sets

After computing full dominator sets, the compiler may compute immediate dominators.

The textbook gives `Idom_Comp()`, which starts from:

```text
Tmp(n) = Domin(n) - {n}
```

Then it removes dominators that are not the closest dominator. The remaining single node is the immediate dominator of `n`.

Example result from the section:

```text
idom(B1)   = entry
idom(B2)   = B1
idom(B3)   = B1
idom(B4)   = B3
idom(B5)   = B4
idom(B6)   = B4
idom(exit) = B1
```

This gives a compact dominator tree:

```text
entry
  └── B1
      ├── B2
      ├── B3
      │   └── B4
      │       ├── B5
      │       └── B6
      └── exit
```

The section states that this bit-vector-style approach can be implemented reasonably efficiently, but its running time is still relatively high: approximately `O(n²e)` for `n` nodes and `e` edges.

---

## Fast Dominator Computation: Lengauer-Tarjan Algorithm

The second algorithm is the faster, more complex method developed by **Lengauer and Tarjan**.

Instead of repeatedly intersecting dominator sets, it uses:

```text
depth-first numbering
semidominators
buckets
path compression
link/eval operations
```

The algorithm first performs a depth-first search and assigns each node a depth-first number. Then it computes a **semidominator** for each node. Informally, the semidominator is an approximation that helps identify the immediate dominator more efficiently.

The key data structures include:

```text
Ndfs(i):     node with depth-first number i
Parent(v):  parent of v in the DFS tree
Sdno(v):    depth-first number of v’s semidominator
Idom(v):    immediate dominator of v
Bucket(v):  nodes grouped by semidominator
Ancestor(v), Label(v): structures used for path compression
```

The algorithm uses `Eval()`, `Link()`, and `Compress()` to maintain auxiliary trees efficiently. With balanced trees and path compression, the running time is:

```text
O(e · α(e, n))
```

where `α` is an extremely slowly growing inverse-Ackermann-like function. Without balanced trees, the algorithm is simpler but runs in:

```text
O(e log n)
```

This makes the Lengauer-Tarjan method much faster than the simple iterative algorithm for large flowgraphs.

---

## Why Dominators Matter for Compiler Optimization

Dominators are not just graph theory; they directly support optimization.

They are used to answer questions like:

```text
Can this block always be reached only after that block?
Can this computation be safely moved upward?
Where is the unique entry of a loop?
Which edge is a loop back edge?
Where should loop-invariant code be placed?
```

The next section uses dominance to define **back edges** and **natural loops**. A back edge is an edge whose head dominates its tail. That definition depends directly on the dominator relation introduced here.

Dominators also appear later in SSA construction, dominance frontiers, control dependence, and program-dependence graphs. In practice, many modern compiler analyses assume that a dominator tree is available.

---

## Key Takeaways

### 1. Dominance means unavoidable control flow

If `d dom i`, every path from `entry` to `i` must pass through `d`.

### 2. Immediate dominators form a tree

The `idom` relation compresses dominance information into a dominator tree, which is easier for compiler algorithms to use.

### 3. Postdominance is dominance in reverse

Postdominators describe nodes that must be reached on every path from a node to `exit`.

### 4. There are two main computation strategies

The simple iterative algorithm is easier to understand but slower. The Lengauer-Tarjan algorithm is harder but much faster for large control-flow graphs.

### 5. Dominators are foundational for later optimizations

Loop recognition, natural loops, SSA form, control dependence, and many code-motion optimizations rely on dominator information.