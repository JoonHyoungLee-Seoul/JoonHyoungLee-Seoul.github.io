---
layout: single
title: "[Compiler] 7.2 Depth-First Search, Preorder Traversal, Postorder Traversal, and Breadth-First Search"
date: 2026-06-09 19:42:32 +0800
categories: Compiler
tags: [Compiler, Control-Flow-Analysis]
toc: true
toc_sticky: true
---

## Core Idea

Section 7.2 introduces four graph traversal concepts used throughout later control-flow analysis algorithms:

- **Depth-first search (DFS)**
    
- **Preorder traversal**
    
- **Postorder traversal**
    
- **Breadth-first search (BFS)**
    

These concepts apply to **rooted directed graphs**, which means they also apply directly to **control-flow graphs (CFGs)**. In compiler optimization, these traversal orders are not just graph theory tools; they determine the order in which CFG nodes are analyzed, classified, or transformed.

---

## Depth-First Search

**Depth-first search** visits a node’s descendants before visiting its unvisited siblings. Starting from the root node, DFS follows one path as deeply as possible, then backtracks.

In a control-flow graph, DFS produces a **depth-first presentation** of the graph. This presentation contains:

1. all graph nodes,
    
2. the edges selected by DFS,
    
3. the remaining non-tree edges classified separately.
    

The selected DFS edges form a **depth-first spanning tree**. These selected edges are called **tree edges**.

### Edge Classification in DFS

DFS classifies non-tree edges into three categories:

|Edge Type|Meaning|
|---|---|
|**Tree edge**|Edge used by DFS to first reach a node|
|**Forward edge**|Edge from a node to one of its descendants, but not a tree edge|
|**Back edge**|Edge from a node to one of its ancestors|
|**Cross edge**|Edge between nodes where neither is an ancestor of the other|

This classification is important because later algorithms, especially loop detection and dominator analysis, depend heavily on understanding ancestor-descendant relationships in the DFS tree.

### Important Point

A depth-first presentation is **not unique**. If a node has multiple successors, different successor visitation orders can produce different DFS trees and different traversal orders.

---

## Generic DFS Routine

The textbook gives a generic DFS routine that supports four customizable processing points:

```text
Process_Before(x)
    run before visiting node x

Process_After(x)
    run after all descendants of x have been visited

Process_Succ_Before(y)
    run before recursively visiting successor y

Process_Succ_After(y)
    run after recursively visiting successor y
```

Conceptually, the DFS structure is:

```text
visit node
    ↓
mark as visited
    ↓
for each successor:
    if unvisited:
        recursively visit successor
    ↓
finish node
```

This generic form is useful because many compiler analyses reuse DFS but attach different actions to these processing points. For example, one algorithm may use DFS to number nodes, while another may use it to classify edges or build a spanning tree.

---

## Preorder Traversal

A **preorder traversal** processes each node **before** its descendants in the DFS spanning tree.

In simple terms:

```text
process current node
    ↓
process children
    ↓
process grandchildren
```

For a CFG, preorder numbering records the order in which nodes are first discovered by DFS.

This is useful because preorder numbers give a compact way to reason about the relative position of nodes in the DFS tree. The textbook’s `Depth_First_Search_PP()` routine assigns a preorder number when a node is first visited.

### Example Intuition

If DFS enters nodes in this order:

```text
entry → B1 → B2 → B3 → exit
```

then this is a preorder traversal because each node is listed when it is first reached.

---

## Postorder Traversal

A **postorder traversal** processes each node **after** its descendants in the DFS spanning tree.

In simple terms:

```text
process children
    ↓
process grandchildren
    ↓
process current node
```

The textbook’s DFS routine assigns a postorder number after all successors have been processed.

### Why Postorder Matters

Postorder is especially important in compiler analysis because many algorithms need to process successors before predecessors.

A common variant is **reverse postorder**, which is widely used in data-flow analysis. In reverse postorder, nodes are processed roughly from entry toward exit, while still respecting much of the CFG’s DFS structure. This often improves convergence speed in iterative data-flow algorithms.

---

## DFS with Preorder, Postorder, and Edge Types

The textbook presents a specialized DFS routine, `Depth_First_Search_PP()`, that computes:

1. a DFS spanning tree,
    
2. preorder numbers,
    
3. postorder numbers,
    
4. edge classifications.
    

The key logic is:

```text
when visiting x:
    assign Pre(x)

for each successor y:
    if y is unvisited:
        recursively visit y
        classify x → y as tree edge
    else if Pre(x) < Pre(y):
        classify as forward edge
    else if Post(y) = 0:
        classify as back edge
    else:
        classify as cross edge

after all successors:
    assign Post(x)
```

This combines traversal and structural classification in one pass. The result is a compact structural summary of the CFG that later algorithms can reuse.

---

## Breadth-First Search

**Breadth-first search** processes all immediate descendants of a node before processing deeper descendants.

DFS goes deep first:

```text
A → B → D → E → C
```

BFS goes level by level:

```text
A → B → C → D → E
```

In a CFG, BFS gives a **breadth-first order**, where nodes closer to the root are numbered before nodes farther away. The textbook’s `Breadth_First()` routine assigns order numbers to successors first, then recursively processes the next level.

### DFS vs. BFS

|Traversal|Main Behavior|Compiler Use|
|---|---|---|
|**DFS**|Explores deeply before backtracking|Dominators, loops, SCCs, structural analysis|
|**Preorder**|Number when first discovered|Node discovery order|
|**Postorder**|Number after descendants finish|Reverse postorder data-flow ordering|
|**BFS**|Explores level by level|Distance-like ordering from root|

---

## Key Takeaways

### 1. DFS Builds Structural Information

DFS is not only a traversal method. In control-flow analysis, it builds a **depth-first spanning tree** and classifies edges into tree, forward, back, and cross edges.

### 2. Traversal Order Affects Later Analyses

Preorder and postorder numbers are used by later algorithms to reason about ancestor-descendant relationships, loop structure, and data-flow iteration order.

### 3. Postorder Is Especially Important for Optimization

Postorder and reverse postorder are useful because they often allow information to propagate through the CFG efficiently.

### 4. BFS Provides a Different View of the CFG

Breadth-first search processes nodes level by level from the root. It is less central than DFS in this chapter’s later algorithms, but it still provides a useful ordering of flowgraph nodes.

### 5. These Traversals Are Building Blocks

Section 7.2 prepares the foundation for the next topics: **dominators, postdominators, loops, strongly connected components, reducibility, and structural analysis**.