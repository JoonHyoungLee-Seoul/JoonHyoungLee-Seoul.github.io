---
layout: single
title: "[Compiler] 12.3 Algebraic Simplifications and Reassociation"
date: 2026-05-21 17:00:10 +0800
categories: Compiler
tags: [Compiler, Optimization]
toc: true
toc_sticky: true
---

> **Source note:** This article is a study note based on Section 12.3 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Algebraic simplification is an early optimization that rewrites expressions using algebraic identities and operator-specific rules. Reassociation is closely related, but its focus is slightly different: it restructures expressions using associativity, commutativity, and distributivity so that the compiler can separate constant terms, loop-invariant terms, and truly variable terms.

In practice, this optimization is not only about making one expression shorter. Its real value is that it exposes cleaner expression forms for later optimizations.

```text
Original expression
    ↓
Algebraic simplification
    ↓
Canonical expression form
    ↓
Reassociation
    ↓
More opportunities for later passes
```

Typical follow-up optimizations include:

```text
Constant propagation
Loop-invariant code motion
Induction-variable recognition
Strength reduction
Dead-code elimination
```

Like constant folding, algebraic simplification and reassociation are best implemented as reusable helper routines that other optimization passes can call whenever they expose new simplification opportunities.

---

## Basic Algebraic Simplification

The most direct simplifications come from identity elements and annihilating operands.

For integer-valued expressions, common examples include:

```text
i + 0  →  i
0 + i  →  i
i - 0  →  i
0 - i  → -i

i * 1  →  i
1 * i  →  i
i / 1  →  i

i * 0  →  0
0 * i  →  0
```

Unary and mixed unary/binary simplifications are also possible:

```text
-(-i)      →  i
i + (-j)  →  i - j
```

Boolean and bit-field expressions have similar simplification rules. For example:

```text
b || true   →  true
b || false  →  b

b && true   →  b
b && false  →  false
```

For bit fields and shifts, the compiler can simplify expressions such as:

```text
f shl 0   →  f
f shr 0   →  f
f shra 0  →  f
```

These rules are small individually, but they reduce noise in the intermediate representation and make later optimization passes easier to apply.

### Example — Making an Induction Variable Visible

Consider the following statement inside a loop:

```fortran
i = i + j * 1
```

If `j` is loop-invariant, the expression still may not be recognized cleanly as an induction-variable update because of the unnecessary multiplication.

After algebraic simplification:

```fortran
i = i + j
```

Now the update pattern is much clearer. The compiler can more easily recognize `i` as an induction variable and later apply loop optimizations such as strength reduction or linear-function test replacement.

### Example — Simplification After Constant Propagation

Other optimizations often create new algebraic simplification opportunities.

```c
j = 0;
k = 1 * j;
i = i + k * 1;
```

After constant folding and constant propagation:

```c
j = 0;
k = 0;
i = i;
```

The assignment `i = i` is now useless and can be removed by dead-code elimination.

```text
Constant folding / propagation
    ↓
Algebraic simplification
    ↓
Dead-code elimination
```

This shows why algebraic simplification is often an enabling optimization rather than a standalone performance win.

---

## Reassociation and Canonicalization

Reassociation rewrites the structure of an expression using algebraic laws.

```text
Associativity:
(a + b) + c  →  a + (b + c)

Commutativity:
a + b        →  b + a

Distributivity:
a*c + b*c    →  (a + b) * c
```

The compiler uses these properties to reorganize expressions into forms that are easier to analyze. A major goal is to group together terms with similar optimization properties.

```text
Expression
    ↓
constant terms
loop-invariant terms
variable terms
```

Canonicalization is the process of choosing a standard representation for equivalent expressions. For example, the compiler may decide that constants should always appear first:

```text
x + 3  →  3 + x
```

This reduces the number of rewrite cases the compiler must check. Without canonicalization, the optimizer needs separate rules for:

```text
x + 0
0 + x
```

With canonicalization, one normalized form is enough. The book notes that this can almost halve the number of cases required for recognizing applicable simplifications.

### Why Reassociation Is Useful

Reassociation can expose:

```text
compile-time constants
loop-invariant subexpressions
common subexpressions
larger strength-reduction candidates
```

For example:

```c
((i - j) + (i - j)) + ((i - j) + (i - j))
```

may look like four separate additions/subtractions. Algebraically, it resembles:

```c
4 * i - 4 * j
```

This form may expose multiplication by constants and make loop-invariant terms easier to isolate. However, this kind of transformation is not always legal, because the rewritten expression may overflow differently or evaluate operations in a different order.

---

## Correctness Limits

Algebraic equivalence in mathematics is not always the same as semantic equivalence in a programming language.

A transformation is legal only if it preserves the behavior required by the source language.

Important correctness constraints include:

```text
integer overflow behavior
floating-point rounding
floating-point exceptions
signed zero
NaN behavior
required evaluation order
parenthesized expression order
```

For example, a reassociation that is harmless in one language may be invalid in another. The text points out that overflow may not matter in C or Fortran 77 in some cases, but it does matter in Ada. It also notes that Fortran 77 requires parenthesized expression evaluation order to be respected, which can make some reassociations invalid even when overflow itself is ignored.

This is the key engineering principle:

```text
Do not optimize based only on algebra.
Optimize based on the source language semantics.
```

---

## 12.3.1 Algebraic Simplification and Reassociation of Addressing Expressions

Addressing expressions are a special case where algebraic simplification and reassociation are especially useful.

The reason is that overflow usually does not matter in address arithmetic. Because of that, the compiler can apply integer-style simplifications more aggressively when computing addresses. The main benefit is that address expressions can be reorganized to expose compile-time constants, loop-invariant expressions, and larger strength-reduction opportunities.

### Example — Two-Dimensional Array Access

Consider a Pascal-style array:

```pascal
var a: array[lo1..hi1, lo2..hi2] of eltype;
i, j: integer;

do j = lo2 to hi2 begin
    a[i, j] := b + a[i, j]
end
```

A typical address calculation for `a[i, j]` may be represented as:

```text
base_a + ((i - lo1) * (hi2 - lo2 + 1) + j - lo2) * w
```

Where:

```text
base_a = base address of array a
lo1    = lower bound of first dimension
hi2    = upper bound of second dimension
lo2    = lower bound of second dimension
w      = element width
```

The compiler tries to transform this expression into a more useful shape.

```text
base_a + ((i - lo1) * (hi2 - lo2 + 1) + j - lo2) * w
    ↓
collect constants
    ↓
collect loop-invariant terms
    ↓
isolate j-dependent part
```

If `j` is the loop variable and `i`, `lo1`, `lo2`, `hi2`, and `w` are loop-invariant, the compiler can separate the address into:

```text
stable base part + j * w
```

The stable part can be computed outside the loop, while the `j * w` part can often be strength-reduced into an incrementing address update.

```text
Before:
    recompute full address expression each iteration

After:
    precompute loop-invariant part
    update address by element width each iteration
```

### Address Simplification Pipeline

The book describes the strategy as converting MIR expressions into trees, recursively applying transformation rules, and then translating the simplified tree back into MIR.

```text
MIR address expression
    ↓
expression tree
    ↓
canonicalization
    ↓
sum-of-products form
    ↓
collect constants and loop-invariants
    ↓
simplified MIR expression
```

### Pointer Expression Simplification

Address-related simplifications are not limited to array indexing. Pointer expressions can also be simplified.

For example, in C:

```c
*(&p)  →  p
```

For a pointer to a structure field:

```c
(&q)->s  →  q.s
```

These transformations remove unnecessary address-taking and dereference operations, producing cleaner IR for later passes.

### Connection to Strength Reduction

Address reassociation often prepares expressions for strength reduction.

```c
addr = base + j * w;
```

Inside a loop over `j`, this may become:

```c
addr = base;

loop:
    use addr;
    addr = addr + w;
```

This replaces repeated multiplication with repeated addition.

```text
Reassociation
    ↓
separate loop-varying address component
    ↓
strength reduction
    ↓
cheaper loop body
```

This is why addressing-expression simplification is one of the most practically important parts of reassociation.

---

## 12.3.2 Application of Algebraic Simplification to Floating-Point Expressions

Floating-point expressions require much more care than integer or address expressions. The book emphasizes that algebraic simplifications are rarely safe for floating-point computations.

The reason is that floating-point arithmetic is not real-number arithmetic. It includes rounding, signed zero, infinities, NaNs, and exceptions.

### Signed Zero

IEEE-style floating-point arithmetic distinguishes between positive and negative zero:

```text
+0.0
-0.0
```

This means apparently simple rewrites may be invalid.

For a positive finite `x`:

```text
x / +0.0  →  +∞
x / -0.0  →  -∞
```

So replacing or removing zero-related computations can change the result.

### NaN and Exceptions

Even this seemingly obvious simplification may be unsafe:

```text
x + 0.0  →  x
```

If `x` is a signaling NaN, evaluating `x + 0.0` may raise an exception, while simply using `x` may not. Therefore, removing the addition can change observable program behavior.

### Reassociation Can Change Results

Let `MF` be the largest finite floating-point value representable in a given precision.

These two expressions can produce different results:

```text
1.0 + (MF - MF)      →  1.0
(1.0 + MF) - MF      →  0.0
```

Mathematically, they may look equivalent. In floating-point arithmetic, they are not equivalent because rounding happens after each operation.

### Example — Machine Epsilon Loop

The book gives a loop like this:

```text
eps := 1.0

while eps + 1.0 > 1.0 do
    oldeps := eps
    eps := 0.5 * eps
od
```

This loop computes the smallest number `x` such that:

```text
1 + x > 1
```

A careless optimizer might rewrite the condition:

```text
eps + 1.0 > 1.0
```

as:

```text
eps > 0.0
```

But this changes the meaning of the loop. The original version computes machine epsilon-like behavior, while the rewritten version computes a much smaller value related to underflow behavior. The book gives the double-precision example where the original result is approximately `2.220446E-16`, while the incorrectly optimized version produces approximately `4.940656E-324`.

### Safe Floating-Point Simplifications

The text cites Farnum’s conservative position: for ANSI/IEEE floating point, only a very small set of algebraic simplifications is generally appropriate. These include:

```text
removing unnecessary type coercions
replacing division by a constant with multiplication by its reciprocal,
but only when both the constant and reciprocal are exactly representable
```

For example, an unnecessary coercion may appear as:

```text
real s
double t

t := (double)s * (double)s
```

On a machine where a single-precision multiply already produces a double-precision result, the explicit conversions may be unnecessary.

For division replacement:

```text
x / C  →  x * (1 / C)
```

This is safe only when both `C` and `1/C` are exactly representable. The ANSI/IEEE inexact flag can help determine whether this condition holds.

---

## Why This Optimization Matters

Algebraic simplification and reassociation are usually not the largest direct source of speedup. Their importance is that they clean up IR and expose structure.

They make later passes more effective by:

```text
removing identity operations
canonicalizing equivalent expressions
exposing constants
enlarging loop-invariant expressions
clarifying induction-variable updates
simplifying address computations
enabling strength reduction
creating dead-code elimination opportunities
```

The most important pattern is:

```text
Small local rewrite
    ↓
Cleaner intermediate representation
    ↓
More precise analysis
    ↓
Stronger downstream optimization
```

This is why the optimization belongs early in the pipeline and why it should be callable from many other passes rather than implemented as a single isolated pass.

---

## Key Takeaways

### 1. Algebraic Simplification Removes Expression Noise

Rules such as `x + 0 → x`, `x * 1 → x`, and `x * 0 → 0` simplify expressions and make the IR easier to analyze.

### 2. Reassociation Restructures Expressions for Optimization

Reassociation uses associativity, commutativity, and distributivity to group constants, loop-invariant terms, and variable terms.

### 3. Canonicalization Reduces Optimizer Complexity

By forcing equivalent expressions into a standard form, the compiler reduces the number of rewrite cases it must recognize.

### 4. Addressing Expressions Are the Most Important Practical Case

Address arithmetic can usually be reassociated aggressively because overflow is not semantically important. This exposes loop-invariant address components and strength-reduction opportunities.

### 5. Floating-Point Expressions Are Dangerous

Floating-point algebraic simplifications are rarely safe because of signed zero, NaNs, exceptions, rounding, infinities, and evaluation-order sensitivity.

### 6. The Main Value Is Enabling Other Passes

Algebraic simplification and reassociation are best understood as cleanup and enabling transformations that improve the effectiveness of later optimizations.