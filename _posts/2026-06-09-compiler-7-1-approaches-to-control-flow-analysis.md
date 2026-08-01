---
layout: single
title: "[Compiler] 7.1 Approaches to Control-Flow Analysis"
date: 2026-06-09
categories: Compiler
tags: [Compiler, Control-Flow-Analysis]
---

> **Source note:** This article is a study note based on Section 7.1 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Big Picture

Control-flow analysis is the compiler’s method for understanding **how execution moves through a procedure**. Before the optimizer can reason about loops, branches, unreachable code, or data-flow facts, it must first recover the control structure of the procedure.

At the source-code level, control structures such as `if`, `for`, and `while` are explicit. After translation into intermediate code, however, these structures often become labels, branches, and jumps. Control-flow analysis rebuilds a more useful representation from that lower-level form.

```text
Intermediate Code
    ↓
Basic Blocks
    ↓
Flowgraph / CFG
    ↓
Dominators, Loops, Intervals
    ↓
Data-Flow Analysis and Optimization
```

The chapter starts from a simple Fibonacci example and shows how a routine is transformed from source code into MIR, then into a flowchart, and finally into a flowgraph.

---

## From Instructions to Basic Blocks

The first major step is to divide a procedure into **basic blocks**.

A basic block is a maximal sequence of instructions that:

- can be entered only at the first instruction,
    
- can be exited only from the last instruction.
    

In other words, a basic block is a straight-line region of code. There are no internal branch targets and no internal exits.

### Leaders

The first instruction of a basic block is called a **leader**. An instruction is a leader if it is:

1. the entry point of the routine,
    
2. the target of a branch,
    
3. the instruction immediately following a branch or return.
    

Once all leaders are found, each leader begins a new basic block. The block continues until the next leader or the end of the routine.

```text
Identify Leaders
    ↓
Split Instructions into Basic Blocks
    ↓
Connect Blocks Using Branches and Fall-Through Edges
```

This basic-block structure is the foundation for every later control-flow analysis method.

---

## Flowgraphs / Control-Flow Graphs

After basic blocks are identified, the compiler constructs a **flowgraph**, also commonly called a **control-flow graph (CFG)**.

In a flowgraph:

```text
Node = basic block
Edge = possible transfer of control
```

The textbook also adds two artificial nodes:

```text
entry
exit
```

The `entry` node points to the initial basic block of the routine. The `exit` node receives edges from blocks that end the routine, such as blocks containing `return`. These nodes are not required by the actual machine code, but they simplify later algorithms, especially data-flow analyses.

A simplified CFG shape looks like this:

```text
entry
  ↓
 B1
 / \
B2 B3
 \ /
 B4
  ↓
exit
```

The compiler then defines:

```text
Succ(b) = set of successor blocks of b
Pred(b) = set of predecessor blocks of b
```

A **branch node** has more than one successor.  
A **join node** has more than one predecessor.

---

## Special Case: Procedure Calls and Nonlocal Control Flow

In most cases, a procedure call does not need to be treated as a basic-block boundary. Treating calls as ordinary instructions creates longer and fewer basic blocks, which is often better for optimization.

However, some calls have unusual control-flow behavior.

The textbook gives C’s `setjmp()` / `longjmp()` as the main example. A normal call returns to the instruction immediately after the call. But with `setjmp()` and `longjmp()`, control may later jump back to the return point of a previous `setjmp()` call from another location.

```text
setjmp point
    ↑
    └── longjmp may transfer control back here
```

This can force the compiler to insert conservative **phantom edges** into the flowgraph. These edges make the analysis safe, but they also make optimization more pessimistic. In practice, compilers may simply avoid optimizing routines that use such constructs.

---

## Dominator-Based Control-Flow Analysis

Section 7.1 introduces two major approaches to control-flow analysis. The first is the **dominator-based approach**.

A node `A` dominates a node `B` if every path from `entry` to `B` must pass through `A`.

```text
A dominates B
=
Every path from entry to B includes A
```

Dominators are useful because they help identify loops. If a flowgraph edge goes from a block back to one of its dominators, that edge is a **back edge**. A back edge usually indicates a loop.

```text
Dominators
    ↓
Back Edges
    ↓
Loop Identification
    ↓
Loop Optimization
```

In the Fibonacci example, the edge from `B6` back to `B4` is a back edge, and the blocks `B4` and `B6` form a loop with `B4` as the loop entry.

This approach is widely used because it is relatively simple to implement and works well with iterative data-flow analysis.

---

## Interval Analysis and Structural Analysis

The second major approach is **interval analysis**.

Instead of only finding loops, interval analysis decomposes the flowgraph into nested regions called **intervals**. These intervals form a hierarchical structure called a **control tree**.

```text
Flowgraph
    ↓
Intervals
    ↓
Nested Regions
    ↓
Control Tree
```

The control tree is useful because it can structure and speed up data-flow analysis. The most sophisticated version of interval analysis is **structural analysis**, which attempts to classify nearly all control-flow structures in a routine.

The textbook explains that dominators with iterative data-flow analysis are common in current optimizing compilers, but interval-based approaches have several advantages:

1. They can make data-flow analysis faster.
    
2. They make it easier to update data-flow information after the program changes.
    
3. Structural analysis makes control-flow transformations easier.
    

So the choice is an engineering tradeoff:

```text
Dominator-based approach:
    easier to implement, widely used

Interval / structural approach:
    more structured, often faster, better for transformations
```

---

## Extended Basic Blocks

A basic block is sometimes too small for optimization. Therefore, the textbook introduces **extended basic blocks**.

An extended basic block is a maximal sequence of instructions beginning with a leader and containing no join nodes except possibly the first node. Conceptually, it is a single-entry tree of basic blocks.

```text
Basic Block:
    one straight-line sequence

Extended Basic Block:
    single-entry tree of basic blocks
```

Example:

```text
    B1
   /  \
 B2    B3
```

If `B2` and `B3` can only be reached through `B1`, then these blocks may form an extended basic block rooted at `B1`.

Extended basic blocks are useful because some local optimizations become more effective when applied to a region larger than a single basic block. The textbook specifically mentions instruction scheduling as an example.

The algorithms `Build_Ebb` and `Build_All_Ebbs` construct extended basic blocks from the flowgraph using successor and predecessor information. In the example flowgraph, the discovered extended basic blocks include `{entry}`, `{B1, B2, B3}`, `{B4, B6}`, `{B5, B7}`, and `{exit}`.

---

## Reverse Extended Basic Blocks

The section also defines **reverse extended basic blocks**.

A normal extended basic block focuses on a single-entry region. A reverse extended basic block is the reverse idea: it is a maximal sequence of instructions ending with a branch node and containing no other branch nodes except the last one.

```text
Extended Basic Block:
    useful for forward, single-entry reasoning

Reverse Extended Basic Block:
    useful for backward, branch-ending reasoning
```

This matters because compiler analyses are not always forward analyses. Some analyses, such as liveness analysis, reason backward through the CFG.

```text
Forward analysis:
    What definitions reach this point?

Backward analysis:
    What values are live before this point?
```

Reverse extended basic blocks provide a larger unit for analyses or optimizations that naturally move backward.

---

## Key Takeaways

Control-flow analysis converts a flat sequence of intermediate-code instructions into a structured graph representation.

The basic workflow is:

```text
Instructions
    ↓
Basic Blocks
    ↓
Flowgraph
    ↓
Dominators / Intervals / Extended Basic Blocks
```

The dominator-based approach is practical, common, and sufficient for many optimizers that use iterative data-flow analysis.

Interval analysis and structural analysis provide a more hierarchical view of control flow and can make data-flow analysis and control-flow transformations more efficient.

Extended basic blocks and reverse extended basic blocks enlarge the scope of local optimization while preserving useful structural constraints.

Overall, Chapter 7 builds the structural foundation needed for Chapter 8’s data-flow analysis and for later optimizations involving loops, branches, scheduling, and control-flow transformations.
