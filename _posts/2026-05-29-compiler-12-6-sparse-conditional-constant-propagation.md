---
layout: single
title: "[Compiler] 12.6 Sparse Conditional Constant Propagation"
date: 2026-05-29
categories: Compiler
tags: [Compiler, Optimization]
---

> **Source note:** This article is a study note based on Section 12.6 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

**Sparse Conditional Constant Propagation (SCCP)** is an optimization that discovers variables whose values are constant and substitutes those constants at their uses. Unlike ordinary constant propagation, SCCP also reasons about **which control-flow paths are executable**.

The optimization therefore combines two effects:

1. **Constant propagation**: replace a variable with a known constant.
2. **Unreachable-code discovery**: eliminate branches and blocks that cannot execute once branch conditions become constant.

### Simple Motivation

Before propagation:

```text
b ← 3
c ← a + b
if b < 4 goto L1 else L2
```

After constant propagation:

```text
b ← 3
c ← a + 3
if 3 < 4 goto L1 else L2
```

After constant-expression evaluation and unreachable-code elimination:

```text
b ← 3
c ← a + 3
goto L1
```

SCCP is more powerful than simply replacing `b` with `3`: it also determines that the edge to `L2` is not executable.

---

## Why Constant Propagation Matters

Constant propagation improves generated code directly and also exposes opportunities for later optimizations.

For RISC-style architectures, small integer constants are especially useful because many instructions can encode small immediate operands directly. Propagating a constant into its use may therefore:

- remove a register requirement,
- avoid an extra load or move instruction,
- simplify address calculations,
- enable constant folding,
- expose induction-variable simplifications,
- improve later dependence-based transformations.

### Optimization Chain

```text
Known Constant Value
        ↓
Constant Propagation
        ↓
Constant-Expression Evaluation
        ↓
Dead or Unreachable Code Removal
        ↓
Simpler and Cheaper Generated Code
```

Ordinary constant propagation only follows data values. SCCP additionally uses constant branch outcomes to determine that some control-flow paths can never be executed.

---

## Why SCCP Uses SSA Form

SCCP is performed on code in **Static Single-Assignment (SSA) form**. In SSA form, every variable has exactly one definition, and each use can be connected directly to its unique defining instruction.

Before running SCCP, the compiler performs the following preparation:

```text
Control-Flow Graph
        ↓
Minimal SSA Transformation
        ↓
Split Basic Blocks into One-Operation Nodes
        ↓
Add SSA Edges from Definitions to Uses
        ↓
Run SCCP Symbolic Execution
```

### Role of SSA Edges

A normal control-flow edge indicates possible execution order:

```text
Block A ──control flow──> Block B
```

An SSA edge transmits value information directly:

```text
definition of x ──SSA value flow──> use of x
```

This is what makes SCCP **sparse**. Instead of repeatedly propagating information through every point in the flowgraph, the algorithm processes only:

- executable control-flow edges, and
- SSA edges whose source values have changed.

As a result, information is transmitted directly from each definition to its relevant uses.

---

## Symbolic Execution and Executable Paths

SCCP is described as a form of **symbolic execution** rather than conventional data-flow analysis.

The algorithm does not assume that every branch of the control-flow graph is reachable. Instead, it gradually marks edges as executable only when the current symbolic values show that execution can follow those edges.

### Branch Handling

Suppose SCCP reaches:

```text
if x < 5 goto Btrue else Bfalse
```

There are three important cases.

### Case 1 — The condition is known to be true

```text
x = 3
3 < 5 = true
```

Only the true edge becomes executable:

```text
        ┌───────┐
        │ x < 5 │
        └───┬───┘
        true│
            ▼
          Btrue
```

### Case 2 — The condition is known to be false

Only the false edge becomes executable.

### Case 3 — The condition is not known to be constant

Both outgoing edges must be considered executable because either path may occur at run time.

This conditional treatment of control flow is the key reason SCCP can discover unreachable code that ordinary constant propagation misses.

---

## The Constant-Propagation Lattice

SCCP associates a lattice value with every SSA variable. The lattice used in the textbook is called `ConstLat`.

```text
                         ⊤
                         │
      false   ...   -1   0   1   ...   true
                         │
                         ⊥
```

The lattice contains:

- `⊤`: the value has not yet been determined; it may still become a constant.
- A specific constant such as `0`, `3`, `false`, or `true`.
- `⊥`: the value is not constant, or cannot be proven constant.

### Meaning of Value Changes

A variable begins at `⊤` and may move downward as more information is discovered:

```text
⊤  →  constant  →  ⊥
```

For example:

```text
⊤ → 3
```

means the variable has been proven to contain the constant value `3`.

```text
3 → ⊥
```

means later information shows that the variable cannot always be `3`.

A critical property is that a lattice cell can be lowered only a small number of times. This limits the total amount of reprocessing required by the algorithm.

### Phi-Function Example

Consider an SSA merge:

```text
x3 ← φ(x1, x2)
```

Possible outcomes include:

```text
x1 = 4, x2 = 4      ⇒ x3 = 4
x1 = 4, x2 = 7      ⇒ x3 = ⊥
x1 = 4, x2 = ⊤      ⇒ x3 may still remain 4 or become less precise later
```

Importantly, SCCP reasons only about values flowing along executable control-flow paths. A constant coming from a path that can never execute should not prevent useful propagation on reachable paths.

---

## Algorithm Data Structures

The textbook algorithm maintains value information and reachability information together.

| Structure | Purpose |
| --- | --- |
| `LatCell(v)` | Stores the current lattice value for SSA variable `v`. |
| `ExecFlag(a, b)` | Records whether flowgraph edge `a → b` has been determined executable. |
| `FlowWL` | Worklist of control-flow edges that need to be processed. |
| `SSAWL` | Worklist of SSA edges whose propagated values may affect uses. |
| `SSASucc(n)` | Returns SSA edges leading from the value defined at node `n` to its uses. |

### Initialization

Initially:

```text
All LatCell values    ← ⊤
All ExecFlag values   ← false
FlowWL                ← edges leaving the entry node
SSAWL                 ← empty
```

At this point, no constants have yet been proven and no branch is considered executable except those discovered from the procedure entry.

---

## The Two-Worklist Algorithm

SCCP uses two interacting worklists because it must propagate two different kinds of information:

```text
FlowWL : executable control-flow information
SSAWL  : SSA value information
```

### High-Level Process

```text
Initialize lattice cells and executable-edge flags

while FlowWL or SSAWL is not empty:

    process an executable control-flow edge
        ↓
    visit the destination instruction or φ-function
        ↓
    possibly update a variable's lattice value
        ↓
    place affected SSA uses into SSAWL
        ↓
    evaluate tests using new lattice values
        ↓
    place newly executable control-flow edges into FlowWL
```

### Processing Control-Flow Edges

When an edge `a → b` is removed from `FlowWL`:

1. If the edge has not previously been marked executable, SCCP marks it executable.
2. If node `b` contains a `φ`-function, the algorithm reevaluates the merge.
3. Otherwise, when the instruction is now reachable under the algorithm's conditions, SCCP evaluates the ordinary instruction.

### Processing SSA Edges

When an SSA edge is removed from `SSAWL`, the destination use is revisited because one of its operands may now have a more precise value.

For example:

```text
a1 ← 2
c1 ← a1 + 3
```

Once SCCP determines:

```text
LatCell(a1) = 2
```

the SSA edge from the definition of `a1` to its use in `c1` causes the second instruction to be reevaluated:

```text
LatCell(c1) = 5
```

---

## Visiting Instructions and Phi-Functions

### Visiting an Ordinary Instruction

For an assignment:

```text
x ← expression
```

SCCP evaluates `expression` using the current lattice values of its operands.

Example:

```text
a1 ← 2
b1 ← 3
c1 ← a1 + b1
```

Once `a1` and `b1` are known:

```text
LatCell(a1) = 2
LatCell(b1) = 3
LatCell(c1) = 5
```

If `LatCell(c1)` changes, all SSA uses of `c1` are scheduled for reconsideration.

### Visiting a Conditional Instruction

For a conditional branch:

```text
if condition goto BY else BN
```

SCCP evaluates the condition in the lattice.

```text
condition = true      ⇒ mark only edge to BY executable
condition = false     ⇒ mark only edge to BN executable
condition = ⊥         ⇒ mark both edges executable
```

A condition still at `⊤` has not yet provided enough information to make a definitive control-flow decision.

### Visiting a Phi-Function

A `φ`-function merges values reaching a control-flow join:

```text
x3 ← φ(x1, x2)
```

The result is computed by combining the lattice values of the incoming definitions. This allows constants discovered along reachable paths to flow through merge points while still safely degrading to `⊥` when conflicting executable values meet.

---

## Example Effect: Discovering Unreachable Code

The first textbook example demonstrates that SCCP can preserve a constant value by proving that a conflicting assignment occurs only on a non-executable path.

Conceptually, consider:

```text
B1: a1 ← 2
        ↓
B3: if a1 < b1 then B4 else B5

B4: c1 ← 4
B5: c2 ← another value

B6: c3 ← φ(c1, c2)
```

Suppose SCCP determines:

```text
LatCell(a1) = 2
LatCell(b1) = 3
a1 < b1 = true
```

Then only the edge to `B4` is executable. The branch to `B5` is never taken, so `c2` does not interfere with the constant result from `B4`.

The resulting information includes:

```text
LatCell(a1) = 2
LatCell(b1) = 3
LatCell(c1) = 4
LatCell(c3) = 4
```

The non-executable branch can then be removed.

### Transformation Pattern

```text
Constant Branch Condition
        ↓
One Successor Edge Becomes Non-Executable
        ↓
Definitions on That Path Become Irrelevant
        ↓
Phi-Functions Become More Precise
        ↓
More Constants and Dead Blocks Are Discovered
```

This interaction between value propagation and reachability is the defining strength of SCCP.

---

## Larger Example and Resulting Simplification

The second textbook example begins with an ordinary program, converts it to minimal SSA form with one operation per node, and then propagates constants through both SSA edges and executable flowgraph edges.

The analysis determines several constant values, including:

```text
a1 = 3
d1 = 2
d3 = 2
a3 = 3
f1 = 5
g1 = 5
a2 = 3
f2 = 6
d2 = 2
```

One value, `f3`, becomes non-constant:

```text
f3 = ⊥
```

After replacing variables with their known constant values and removing unreachable code, the resulting flowgraph is substantially smaller. Figure 12.36 in the textbook illustrates this simplified result: SCCP preserves only the executable nodes and removes branches whose conditions have been resolved symbolically.

The example demonstrates that SCCP does more than evaluate isolated arithmetic expressions. It can simplify an entire procedure by repeatedly feeding constant information into control-flow decisions.

---

## Basic Blocks Versus Single-Instruction Nodes

For presentation, the textbook formulates the algorithm using flowgraph nodes that contain one statement each. This makes the relationship between SSA definitions, SSA uses, and executable flowgraph edges easier to describe.

However, this representation is not essential.

A practical compiler can apply the same algorithm to ordinary basic blocks, provided that definition sites are identified precisely, for example by:

```text
(block number, instruction position within the block)
```

Thus, SCCP is directly applicable to realistic compiler IRs in which blocks contain multiple instructions.

---

## Complexity and Practical Efficiency

The time complexity of SCCP is bounded by the number of control-flow edges plus the number of SSA edges:

```text
O(|E| + |SSA|)
```

where:

- `E` is the set of control-flow edges,
- `SSA` is the set of SSA definition-use edges.

The reason is that each SSA variable's lattice value can be lowered only twice:

```text
⊤ → constant → ⊥
```

In the worst case, the number of edges may be quadratic in the number of nodes. In practice, however, control-flow graphs and SSA use graphs are usually sparse, so SCCP behaves almost linearly.

### Why It Is Efficient

```text
Classic propagation:
    repeatedly inspect broad regions of the control-flow graph

SCCP:
    process only executable flow edges
    process only affected SSA uses
    update each lattice value only a bounded number of times
```

This combination of sparse value propagation and conditional reachability analysis gives SCCP both strong optimization power and good practical performance.

---

## Relationship to Other Early Optimizations

SCCP appears in the early optimization phase because it can simplify the program before more expensive transformations are applied.

Its effectiveness is closely connected to other passes:

```text
Scalar Replacement
        ↓
SCCP
        ↓
Constant Folding
        ↓
Dead-Code Elimination
        ↓
Value Numbering / Redundancy Elimination
```

For example:

- scalar replacement may expose scalar constant values hidden inside aggregates;
- SCCP may resolve branch conditions;
- constant folding evaluates newly formed constant expressions;
- dead-code elimination removes unreachable or unused computations;
- later optimizations operate on a smaller and simpler program.

Because earlier transformations can expose new constants, and SCCP can expose new dead code or new scalarization opportunities, an aggressive optimizer may run SCCP more than once.

---

## Key Takeaways

### 1. SCCP Combines Value Analysis with Reachability Analysis

Ordinary constant propagation replaces known variables with constants. SCCP additionally determines which control-flow edges are executable, allowing it to remove unreachable paths.

### 2. SSA Form Makes Propagation Sparse

Each SSA definition is connected directly to its uses. Constant information is propagated only where it is relevant rather than throughout the entire control-flow graph.

### 3. The Lattice Represents Increasing Knowledge

Variables begin at `⊤`, may become a specific constant, and may eventually degrade to `⊥` when they cannot be proven constant.

### 4. Two Worklists Coordinate Control Flow and Data Flow

`FlowWL` discovers reachable execution paths, while `SSAWL` transmits changed constant values through SSA definition-use relationships.

### 5. Conditional Information Improves Phi-Function Precision

If one predecessor of a merge is unreachable, its value does not destroy a constant coming from the executable predecessor. This allows SCCP to discover constants that traditional propagation would miss.

### 6. SCCP Is Both Powerful and Efficient

Its practical running time is nearly linear because each control-flow and SSA edge is processed only as needed, and each lattice value can descend only a bounded number of times.
