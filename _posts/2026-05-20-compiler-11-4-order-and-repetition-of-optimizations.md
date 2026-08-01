---
layout: single
title: "[Compiler] 11.4 Order and Repetition of Optimizations"
date: 2026-05-20 08:46:07 +0800
categories: Compiler
tags: [Compiler, Optimization]
toc: true
toc_sticky: true
---


> Compiler optimization is not only about _which optimizations exist_,  
> but also about _when_ and _how often_ they should run.

## The Core Problem: Phase Ordering

Modern compiler optimizations are deeply interconnected.

One optimization may:

- expose opportunities for another optimization,
- remove opportunities for another,
- or only become effective after previous transformations.

This creates one of the hardest problems in compiler engineering:

## Phase Ordering Problem

```text
Optimization A
    ↓
creates simplification opportunities
    ↓
Optimization B becomes effective
    ↓
new dead code appears
    ↓
Optimization C removes it
    ↓
Optimization A becomes useful again
```

The textbook explicitly notes that:

> “No order can be optimal for all programs.”

Different:

- architectures,
- workloads,
- optimization goals,

all prefer different optimization pipelines.

---

## Why Optimization Ordering Matters

Consider the following example.

### Example — Inlining Before Constant Propagation

#### Original Code

```cpp
int add3(int x) {
    return x + 3;
}

int main() {
    int y = add3(5);
}
```

#### Without Inlining

The compiler sees:

```text
main() → unknown function call
```

At this stage:

- constant propagation cannot occur,
    
- because the function boundary hides information.
    

#### After Inlining

```cpp
int y = 5 + 3;
```

Now multiple optimizations become possible:

- Constant Folding
- Dead Code Elimination
- Branch Simplification    

Final result:

```cpp
int y = 8;
```

#### Optimization Chain

```text
Inlining
    ↓
constants exposed
    ↓
Constant Folding
    ↓
Dead Code Elimination
```

Inlining itself was not the final optimization.

Its real value was:

> exposing optimization opportunities for later passes.

---

## High-Level Optimization Pipeline

The book proposes a general optimization ordering strategy:

```text
High-Level IR (HIR)
    ↓
[A] Array / Loop / Cache Optimizations
    ↓
[B] Early Scalar & Interprocedural Optimizations
    ↓
[C] Global Dataflow Optimizations
    ↓
[D] Low-Level Machine Optimizations
    ↓
[E] Link-Time Optimizations
```

Each stage operates at a different abstraction level.

### A — High-Level Optimizations

These optimizations require:

- explicit loops,
- array indexing,
- high-level memory structure.

Typical examples:

- Scalar Replacement of Arrays
- Data Cache Optimization
- Dependence Analysis

#### Why Scalar Replacement Comes First

The book recommends performing scalar replacement before cache optimization.

Why?

```text
Array references
    ↓
converted into scalar temporaries
    ↓
fewer memory accesses remain
    ↓
cache optimization becomes easier
```

#### Example

Before:

```cpp
for (int i = 0; i < N; i++)
    sum += A[i];
```

After scalar replacement:

```cpp
tmp = A[i];
sum += tmp;
```

The compiler can now:

- reuse registers,
- reduce loads,
- simplify later analyses.

---

### B — Early / Interprocedural Optimizations

These passes are executed early because they expose large optimization opportunities.

Typical examples:

- Inlining
- Tail Recursion Elimination
- SCCP
- Interprocedural Constant Propagation
- Procedure Specialization

#### Tail Recursion Elimination

Before:

```cpp
int fact(int n, int acc) {
    if (n == 0)
        return acc;

    return fact(n - 1, n * acc);
}
```

After optimization:

```cpp
while (n != 0) {
    acc = n * acc;
    n--;
}
```

Benefits:

- removes recursive calls,
- eliminates stack overhead,
- enables loop optimizations.

---

## Why SCCP Is Repeated

This is one of the most important ideas in the chapter.

### SCCP — Sparse Conditional Constant Propagation

SCCP combines:

- constant propagation,
- branch analysis,
- CFG simplification.

It tracks:

- which values are constant,
- which branches are executable.

### Example

#### Before Procedure Specialization

```cpp
foo(mode, x);
```

#### After Procedure Specialization

```cpp
foo_mode1(x);
```

Now the compiler knows:

```cpp
mode == 1
```

is always true.

### New Optimization Opportunities

The compiler can now remove:

```cpp
if (mode == 1)
```

This enables:

- branch folding,
- dead code elimination,
- CFG simplification.

### Why SCCP Runs Again

```text
Procedure Specialization
    ↓
new constants appear
    ↓
SCCP becomes profitable again
```

Optimization is therefore iterative.

---

## Optimization Is Not “One Pass”

Real compiler optimization usually looks like this:

```text
Optimize
    ↓
Simplify
    ↓
Expose opportunities
    ↓
Optimize again
```

not:

```text
Run one pass once → finished
```

### Real LLVM Example

LLVM optimization pipelines often look like:

```text
InstCombine
SimplifyCFG
DCE
InstCombine
GVN
LICM
SimplifyCFG
InstCombine
```

This repetition is intentional.

### Why Repetition Works

```text
InstCombine
    ↓
creates dead instructions
    ↓
DCE removes them
    ↓
CFG becomes simpler
    ↓
InstCombine simplifies more
```

Each pass creates opportunities for the next pass.

---

## C — Global Dataflow Optimizations

These optimizations rely heavily on dataflow analysis.

Typical examples:

- Common Subexpression Elimination (CSE)
- Partial Redundancy Elimination (PRE)
- Loop Invariant Code Motion (LICM)

Required analyses:

- reaching definitions,
- liveness,
- redundancy analysis.

---

## D — Low-Level Machine Optimizations

Performed near machine code generation.

Typical examples:

- Register Allocation
- Instruction Scheduling
- Software Pipelining
- Branch Scheduling

These optimizations are:

- architecture-aware,
- latency-aware,
- register-pressure-aware.

---

## Why IR Level Matters

Different optimizations require different abstraction levels.

### High-Level IR

Needed for:

- dependence analysis,
- array optimization,
- cache optimization.

The compiler still needs:

- loop structure,
- multidimensional indexing,
- semantic information.

### Low-Level IR

Needed for:

- instruction scheduling,
- register allocation,
- machine idiom recognition.

The compiler now needs:

- actual instructions,
- register constraints,
- hardware latency information.

---

## Optimizations Cooperate

Compiler passes are not isolated modules.

They form an optimization graph:

```text
Inlining
    ↓
Constant Propagation
    ↓
Dead Code Elimination
    ↓
CFG Simplification
    ↓
PRE / LICM improve
```

This interaction is the foundation of modern compiler pipelines.

---

## Key Insight

Optimization quality is often determined less by:

- individual optimization algorithms,

and more by:

- pass ordering,
- repetition strategy,
- IR organization,
- fixed-point convergence behavior.

This is why modern compilers spend enormous engineering effort on:

- optimization pipelines,
- pass scheduling,
- profitability heuristics,
- iterative cleanup loops.

---

## Connection to Modern Compiler Design

This chapter directly explains the philosophy behind:

- LLVM optimization pipelines,
- GCC pass managers,
- MLIR pass scheduling,
- iterative cleanup pipelines.

Optimization passes are valuable not only for what they optimize directly,  
but also for the optimization opportunities they expose for other passes.

---

## Key Takeaways

### 1. Optimization Order Matters

Different pass orderings produce different generated code.

### 2. Optimizations Create Opportunities for Each Other

Compiler optimizations are cooperative, not isolated.

### 3. Many Optimizations Must Be Repeated

Especially:

- SCCP
- InstCombine
- CFG Simplification
- DCE

### 4. IR Level Determines Optimization Capability

- High-level IR → semantic optimizations
- Low-level IR → machine-aware optimizations
    

### 5. Phase Ordering Is One of the Hardest Compiler Problems

Even modern production compilers do not solve it perfectly.