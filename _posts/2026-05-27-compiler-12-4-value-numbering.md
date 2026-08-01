---
layout: single
title: "[Compiler] 12.4 Value Numbering"
date: 2026-05-27 08:32:55 +0800
categories: Compiler
tags: [Compiler, Optimization]
toc: true
toc_sticky: true
---

Value numbering is an optimization technique for discovering computations that produce the same value, even when they do not appear as textually identical expressions. Once two computations are known to represent the same value, a later computation can often be replaced by a copy of the already computed result.

This section presents two forms of value numbering:

- **Local value numbering**, which operates within one basic block using hashing.
    
- **Global value numbering**, which operates across an entire procedure using SSA form, value graphs, and congruence-class partitioning.

---

## Core Idea: Number Values, Not Expressions

Traditional redundancy elimination often asks:

> Has this exact expression already been computed?

Value numbering asks a stronger question:

> Has a computation with the same resulting value already been computed?

Consider:

```text
j ← i + 1
k ← i
l ← k + 1
```

Even though `i + 1` and `k + 1` are not textually identical, value numbering can infer:

```text
k ≡ i
therefore
k + 1 ≡ i + 1
therefore
l ≡ j
```

The compiler may then rewrite:

```text
l ← j
```

This is the central idea:

```text
Different syntax
    ↓
Same abstract value
    ↓
Replace recomputation with reuse
```

A **value number** is an abstract identifier assigned to a computed value. Expressions that are proven equivalent receive the same value number.

---

## Relationship to Other Optimizations

Value numbering overlaps with constant propagation and common-subexpression elimination, but none of them completely subsumes the others.

### Case 1 — Value Numbering Finds Equality That Constant Propagation Cannot

```text
read(i)
j ← i + 1
k ← i
l ← k + 1
```

Value numbering determines that `j` and `l` contain the same value because `k` is a copy of `i`.

Constant propagation cannot do this because `i` is not known to be a constant. Common-subexpression elimination also does not directly expose the equivalence because `i + 1` and `k + 1` are syntactically different.

### Case 2 — Constant Propagation Finds Equality That Value Numbering Cannot

```text
i ← 2
j ← i * 2
k ← i + 2
```

Constant propagation can evaluate both expressions:

```text
j = 4
k = 4
```

Basic value numbering does not generally interpret arithmetic identities such as:

```text
2 * 2 = 2 + 2
```

Therefore, it may not prove `j` and `k` equivalent.

### Case 3 — Redundancy Elimination Finds Opportunities That Value Numbering Misses

```text
read(i)
l ← 2 * i
if i > 0 goto L1
j ← 2 * i
goto L2
L1:
k ← 2 * i
L2:
```

Global common-subexpression elimination or partial-redundancy elimination can determine that later computations of `2 * i` are redundant along their execution paths.

Value numbering alone may not eliminate them because the values arriving from different control-flow paths are not necessarily represented as one congruent value.

The comparison in Figure 12.11 establishes an important compiler-engineering point:

```text
Value Numbering      Constant Propagation      Redundancy Elimination
        \                    |                         /
         \                   |                        /
          Each exposes different optimization opportunities
```

---

## 12.4.1 Local Value Numbering

Local value numbering operates on one **basic block** at a time. Since a basic block has only one entry and no internal branch targets, instructions can be processed sequentially without requiring global data-flow analysis.

### Goal

Within a basic block, local value numbering attempts to:

- detect repeated computations,
- recognize equivalent expressions such as commuted operands,
- rewrite redundant computations as copies,
- expose expressions inside non-assignment instructions,
- invalidate expressions whose operand values have been redefined.

### Basic Example

Suppose a basic block contains:

```text
a ← i + 1
b ← 1 + i
```

Because addition is commutative:

```text
i + 1 ≡ 1 + i
```

the second computation is redundant:

```text
a ← i + 1
b ← a
```

The value is computed once and reused.

### Why This Is More Than Textual Matching

A purely syntactic matcher would treat:

```text
i + 1
1 + i
```

as different expressions.

Value numbering may normalize commutative operations so that both expressions map to the same equivalence class:

```text
Hash(i + 1) = Hash(1 + i)
```

This allows equivalent computations to be recognized even when operand order differs.

---

## Hash-Based Local Value Numbering

The local algorithm uses hashing to organize expressions into candidate equivalence classes.

### High-Level Procedure

For each instruction in the basic block:

```text
Compute expression hash
        ↓
Search previously recorded expressions in the same bucket
        ↓
Equivalent expression already exists?
       / \
     Yes  No
      |    |
Replace   Record the new
with use  expression
```

More concretely:

1. Compute a hash value for the expression.
    
2. Look in the hash bucket for previous expressions with the same hash.
    
3. Compare expressions in that bucket for actual equivalence.
    
4. If a match is found, replace the current computation with a use of the prior result.
    
5. If no match is found, insert the new expression into the bucket.
    

Hashing does not itself prove equivalence. It merely narrows the search to likely matches.

### Example

```text
1: a ← x ∨ y
2: b ← x ∨ y
```

After processing instruction 1, the value of `x ∨ y` is recorded as available through `a`.

When instruction 2 is processed, its expression matches the expression already associated with `a`, so the block becomes:

```text
1: a ← x ∨ y
2: b ← a
```

The second computation has been converted into a copy.

---

## Handling Expressions Inside Control-Flow Instructions

An expression does not always appear on the right-hand side of an assignment. For example:

```text
if !z goto L1
```

If `!z` later appears again, the compiler needs a name for its value in order to reuse it.

Local value numbering therefore rewrites an expression-bearing conditional instruction into two instructions:

```text
t1 ← !z
if t1 goto L1
```

Now the computed value is explicit and reusable.

For example:

```text
if !z goto L1
x ← !z
```

can become:

```text
t1 ← !z
if t1 goto L1
x ← t1
```

This transformation is illustrated in Figure 12.15, where an expression initially embedded in an `if` instruction is assigned to a temporary and subsequently reused.

---

## Killed Expressions and Redefinition

A previously computed expression remains reusable only while the values of its operands remain unchanged.

Consider:

```text
a ← x ∨ y
x ← !z
b ← x ∨ y
```

The first computation of `x ∨ y` cannot be reused for `b`, because `x` has been redefined.

```text
Before redefining x:
    x ∨ y is available

After redefining x:
    old x ∨ y is invalid
```

The local algorithm therefore performs a **kill step** whenever a variable is assigned a new value:

```text
x ← ...
```

All recorded expressions that use `x` as an operand must be removed from the hash structure.

### Role of `Remove()`

Figure 12.14 provides a routine that scans hash buckets and deletes stored expressions whose operands contain the redefined variable.

Conceptually:

```text
Assignment to variable v
        ↓
Remove every available expression containing v
        ↓
Prevent incorrect reuse of stale values
```

This is essential for correctness. Value numbering is not merely finding similarities; it must track whether previously computed values are still valid.

---

## Worked Local Transformation

Figure 12.15 demonstrates the behavior of local value numbering on a basic block containing repeated Boolean expressions and a redefinition.

A simplified form of the transformation is:

### Before Optimization

```text
a ← x ∨ y
b ← x ∨ y
if !z goto L1
x ← !z
c ← x ∧ y
if x ∧ y trap 30
```

### After Local Value Numbering

```text
a ← x ∨ y
b ← a

t1 ← !z
if t1 goto L1
x ← t1

c ← x ∧ y
if c trap 30
```

### What Happened

```text
x ∨ y computed for a
    ↓
same expression reused for b

!z appears inside a branch
    ↓
materialized into temporary t1

x receives a new value
    ↓
expressions depending on the old x are killed

x ∧ y computed for c
    ↓
same value reused in the trap condition
```

This example captures the operational behavior of local value numbering:

- discover equivalence,
    
- materialize unnamed computations,
    
- preserve correctness through kills,
    
- replace redundant computations with value reuse.
    

---

## Relationship to Basic-Block DAGs

Local value numbering closely resembles construction of a DAG representation for a basic block.

In a DAG:

```text
Repeated computation
    ↓
Reuse existing node
```

In value numbering:

```text
Repeated computation
    ↓
Reuse previously computed variable
```

For example, two occurrences of:

```text
x + y
```

would correspond to one shared DAG node. In value numbering, the later expression would be replaced with a use of the variable storing the result of the earlier computation.

Therefore:

```text
DAG node reuse  ≈  value-number reuse
```

The book notes that value numbering is frequently used while constructing DAGs for basic blocks.

---

## 12.4.2 Global Value Numbering

Local value numbering is limited to a single basic block. It cannot directly recognize equivalent computations that occur in different blocks or across loops.

**Global value numbering (GVN)** extends the idea to an entire procedure.

The method presented in the book is based on the approach of **Alpern, Wegman, and Zadeck**. It requires the procedure to be represented in **SSA form**.

### Why SSA Form Is Required

In ordinary intermediate code, a variable may receive multiple definitions:

```text
x ← ...
...
x ← ...
```

This makes it difficult to determine which value an occurrence of `x` represents.

SSA form gives each definition a unique name:

```text
x1 ← ...
x2 ← ...
```

and uses `φ`-functions where control-flow paths merge:

```text
x3 ← φ(x1, x2)
```

This provides a clean foundation for reasoning about global value equivalence.

```text
Control-flow graph
    ↓
Minimal SSA form
    ↓
Value graph
    ↓
Congruence analysis
    ↓
Global value equivalence
```

The book obtains minimal SSA form using the iterated-dominance-frontier method introduced earlier in Section 8.11.

---

## Value Graphs

A **value graph** is the central representation used for global value numbering.

### Definition

A value graph is a labeled directed graph in which:

- nodes represent constants, operators, function symbols, or unknown input values,
    
- edges connect an operation to its operands,
    
- edge labels indicate operand position,
    
- SSA variables name the nodes whose values they receive.
    

Consider:

```text
a ← 3
b ← 3
c ← a + 1
d ← b + 1
```

A conceptual value graph is:

```text
        3
       / \
      a   b

      +       +
     / \     / \
    a   1   b   1
    |           |
    c           d
```

Because `a` and `b` both represent the constant `3`, the two addition nodes compute the same value. Thus:

```text
c ≡ d
```

The value graph converts value equivalence into a graph-equivalence problem.

### Cycles in Value Graphs

For straight-line code, the graph may be acyclic. For loops in SSA form, value graphs may contain cycles because loop-carried values depend on `φ`-functions and computations in the loop body.

The examples in Figures 12.18–12.20 show a flowgraph, its minimal SSA representation, and the resulting cyclic value graph used for global analysis.

---

## Congruence of Values

GVN determines equivalence through a relation called **congruence**.

Two value-graph nodes are congruent when one of the following holds:

1. They are the same node.
    
2. They are constant nodes containing the same constant.
    
3. They represent the same operator and their corresponding operands are congruent.
    

For example:

```text
a ≡ b
1 ≡ 1
----------------
a + 1 ≡ b + 1
```

However, congruence alone is not sufficient to replace one variable with another at an arbitrary program point.

### Dominance Requirement

Two variables are equivalent at program point `p` only when:

```text
their value-graph nodes are congruent
and
their defining assignments dominate p
```

This requirement guarantees that the reused value has definitely been computed before it is used.

```text
Same computed value
        +
Definition available on every path
        =
Safe replacement
```

---

## Partitioning Algorithm for Global Value Numbering

The book computes congruence using iterative partition refinement.

### Initial Partitioning

Initially, all nodes with the same label are placed in the same partition.

Examples:

```text
all "+" nodes together
all constant-3 nodes together
all φ-nodes together
```

This initial assumption is intentionally optimistic: nodes with the same operator might compute equal values.

### Refinement

The algorithm then checks the operands of nodes within each partition.

Suppose two `+` nodes initially belong to the same class:

```text
x ← a + b
y ← c + d
```

They remain congruent only when:

```text
a ≡ c
and
b ≡ d
```

Otherwise, the partition must be split.

### Fixed Point

Partitioning continues until no class needs further splitting:

```text
Initial classes by node label
        ↓
Split classes whose operands disagree
        ↓
Repeat splitting
        ↓
No additional split possible
        ↓
Maximal congruence relation
```

At the fixed point, each partition represents a class of computations proven to yield the same value.

---

## Data Structures Used by `Global_Value_Number`

The global algorithm in Figure 12.21 operates on four principal structures.

|Structure|Meaning|
|---|---|
|`N`|Set of nodes in the value graph|
|`NLabel`|Function mapping each node to its label|
|`ELabel`|Labeled operand edges between nodes|
|`B`|Array of partitions representing candidate congruence classes|

The algorithm also maintains a **worklist** of nontrivial partitions that may still require refinement.

### Auxiliary Operations

|Routine|Purpose|
|---|---|
|`Initialize()`|Creates initial partitions based on node labels and initializes the worklist|
|`Arity()`|Returns the number of operands associated with an operator node|
|`Follow_Edge()`|Finds a node’s operand corresponding to a given operand position|

### Complexity

The worst-case running time of the partitioning algorithm is:

```text
O(e log e)
```

where `e` is the number of edges in the value graph.

---

## Example: When Congruence Succeeds and Fails

The global examples demonstrate how structurally similar computations can either remain equivalent or diverge.

### Congruent Case

When corresponding `i` and `j` computations apply equivalent operators to congruent operands, their value-graph nodes remain in the same final partitions:

```text
i-value computation ≡ j-value computation
```

The compiler can therefore recognize global equality between corresponding values.

### Non-Congruent Case

The book then modifies one assignment so that an addition becomes a subtraction:

```text
i ← i - 3
```

instead of:

```text
i ← i + 3
```

Although the overall control-flow structure remains similar, the operator labels differ:

```text
+ ≠ -
```

Partition refinement separates the corresponding `i` and `j` nodes. The apparent structural similarity is no longer enough to prove value equivalence.

This demonstrates why GVN is based on precise operator-and-operand congruence rather than superficial resemblance between computations.

---

## Extensions and Limitations

The global method described in this section can be strengthened in several ways.

### Control-Flow-Sensitive Congruence

Alpern, Wegman, and Zadeck propose incorporating structural analysis and specialized `φ`-functions for control-flow constructs. This allows the analysis to capture value relationships that depend on program structure.

### Arrays Through Functional Modeling

Array updates are difficult because modifying one element changes part of an aggregate object. The book describes a functional representation such as:

```text
a[i] ← 2 * b[i]
```

modeled as:

```text
a ← update(a, i, 2 * access(b, i))
```

This turns array operations into explicit value-producing functions that can participate in a value graph.

### Commutativity

Basic global congruence does not automatically equate:

```text
a * b
b * a
```

Extending the method to account for commutative operators allows more equivalent computations to be recognized.

### Hash-Based and Graph-Based Approaches Are Incomparable

Later work cited in the section extends hash-based value numbering along dominator trees and compares it against the partitioning-based global technique.

The important result is:

```text
Hash-based value numbering is better in some cases.
Partitioning-based global value numbering is better in others.
```

A later approach operating on strongly connected components of an SSA representation combines advantageous properties of both methods and is reported as more effective than either individually.

---

## Compiler Engineering Perspective

Value numbering is important because it turns implicit semantic equality into explicit reusable values.

### Local Form

```text
Basic block
    ↓
Hash expressions
    ↓
Reuse equivalent computations
```

This form is inexpensive and works well for straight-line code.

### Global Form

```text
Procedure in SSA form
    ↓
Construct value graph
    ↓
Partition nodes by congruence
    ↓
Reuse equivalent values across blocks
```

This form is more powerful, but requires additional intermediate representation and analysis infrastructure.

### Placement in an Optimization Pipeline

Value numbering is an early optimization because it simplifies the value relationships seen by later passes:

```text
Value Numbering
    ↓
Copy Propagation
    ↓
Sparse Conditional Constant Propagation
    ↓
Dead-Code Elimination
    ↓
Redundancy Elimination
```

For example, replacing a redundant expression with a copy may allow later copy propagation to eliminate intermediate variables, while exposing equivalent values may help later transformations reason more effectively about control flow and redundant computations.

---

## Key Takeaways

### 1. Value Numbering Identifies Equivalent Results

Value numbering recognizes when different-looking computations produce the same value and replaces unnecessary recomputation with reuse.

### 2. Local Value Numbering Uses Hashing

Within a basic block, expressions are hashed into candidate equivalence classes. Matching expressions are replaced with copies of previously computed values.

### 3. Redefinitions Kill Available Expressions

When an operand variable is assigned a new value, previously recorded expressions depending on that variable must be removed to preserve correctness.

### 4. Global Value Numbering Requires SSA Form

SSA makes each value definition explicit and allows equivalence reasoning across basic blocks and loops.

### 5. Value Graphs Encode Computation Structure

Global value numbering converts the procedure into a graph whose nodes represent values and whose edges represent operand relationships.

### 6. Congruence Is Computed by Partition Refinement

Nodes begin in broad classes based on labels and are repeatedly separated until only genuinely equivalent computations remain together.

### 7. Value Numbering Complements Other Optimizations

It is neither a replacement for constant propagation nor for redundancy elimination. Each analysis discovers opportunities that the others may miss.