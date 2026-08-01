---
layout: single
title: "[Compiler] 4.6 Our Intermediate Languages (MIR, HIR, and LIR)"
date: 2026-06-01
categories: Compiler
tags: [Compiler, Intermediate-Representation]
---

> **Source note:** This article is a study note based on Section 4.6 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

Section 4.6 introduces the three intermediate languages used throughout the book: **MIR**, **HIR**, and **LIR**. The author’s main goal is not to claim that these are universal best IRs, but to provide concrete representations on which later optimization and code-generation algorithms can be explained. The chapter states that **MIR** is the primary representation, **HIR** is used when high-level program structure matters, and **LIR** is used when low-level machine details must be explicit.

---

## Core Idea

The compiler does not use only one intermediate representation. Instead, it uses a small family of related IRs:

```text
Source Program
    ↓
HIR  → high-level structure preserved
    ↓
MIR  → general-purpose optimization form
    ↓
LIR  → machine-oriented low-level form
    ↓
Target Code
```

The three forms are intentionally related. **HIR** and **LIR** are not completely separate languages; they are extensions or adaptations of **MIR**. This allows the book to explain most transformations using one common framework while still switching levels when necessary.

The section explicitly says that most examples use **MIR**, while **HIR** is used for higher-level features such as array subscripts, and **LIR** is used when registers, memory addresses, and similar machine-level details need to be represented directly.

## MIR: Medium-Level Intermediate Representation

**MIR** is the default intermediate language used in the book. It is a medium-level representation designed to be abstract enough for machine-independent optimization, but concrete enough to express procedure bodies, control flow, calls, assignments, pointer operations, and structured data access.

Conceptually, MIR consists of:

- a symbol table,
    
- quadruple-like instructions,
    
- operators,
    
- operands,
    
- temporary variables,
    
- labels,
    
- and a small set of instruction forms.
    

A MIR program is made of program units. Each unit has an optional label and a sequence of instructions delimited by `begin` and `end`. MIR instructions may include `receive`, assignment, `goto`, `if`, `call`, `return`, and sequencing instructions.

### MIR Instruction Style

MIR instructions are often written like assignments:

```text
t1 <- a + b
x  <- t1
```

This makes MIR easy to read as a three-address-code-like language. The important point is that MIR expresses computation in small, explicit steps.

For example, a C procedure call and linked-list manipulation can be lowered into MIR using `receive`, `call`, pointer dereference, field access, conditional branches, and returns. The book’s example translates two C procedures, `make_node` and `insert_node`, into MIR. The MIR version explicitly shows parameter reception, calls to `malloc`, field assignments such as `*q.next <- nil`, and conditional control flow.

### MIR Operators

MIR supports arithmetic, relational, shift, logical, and component-selection operators. Examples include:

```text
+  -  *  /  mod
=  !=  <  <=  >  >=
shl  shr  shra
and  or  xor
*  .
```

It also includes unary operators such as arithmetic negation, logical negation, address-of, type conversion, and pointer indirection.

The key design choice is that MIR still retains useful semantic information. For example, multiplication and division remain visible as operations even if the target machine may not have direct hardware support for them. This allows later optimizations such as algebraic simplification or strength reduction to recognize the operation before final code generation.

## HIR: High-Level Intermediate Representation

**HIR** is a higher-level variant of MIR. It is used when preserving source-level structure helps analysis or optimization.

The main features added by HIR are:

```text
MIR
 + structured for-loops
 + structured if-statements
 + high-level array references
 + trap instruction
```

The most important difference is how HIR handles arrays. In MIR, an array reference may already be lowered into address arithmetic. In HIR, an array access can remain as a structured subscripted reference:

```text
A[i, j]
```

rather than being translated immediately into something like:

```text
base(A) + offset(i, j)
```

This matters because many loop and memory optimizations need to understand the original subscript structure. Dependence analysis, vectorization, parallelization, and data-cache optimizations often depend on knowing that a program is accessing `A[i]`, `A[i+1]`, or `A[j]`, not merely some computed address.

### HIR For-Loops

HIR includes a structured `for` loop form:

```text
for v <- opd1 by opd2 to opd3
    instructions
endfor
```

The book gives the corresponding MIR-style semantics: the loop is lowered into initialization, conditional tests, explicit jumps, body execution, induction-variable update, and loop exit logic.

This shows the role of HIR clearly: it is not a different execution model. It is a more structured representation that can later be translated into lower-level MIR control flow.

### Why HIR Is Useful

HIR is useful when the compiler needs to reason about the program at a level close to the source language.

For example:

```text
for i <- 1 by 1 to n
    A[i] <- A[i] + 1
endfor
```

At the HIR level, the compiler can still see:

```text
loop variable: i
array: A
subscript: i
loop bounds: 1 to n
```

That information is much harder to recover after everything has been lowered into pointer arithmetic and branches.

## LIR: Low-Level Intermediate Representation

**LIR** is the low-level adaptation of MIR. It is used when the compiler must expose machine-oriented details such as registers and memory addresses.

The section describes LIR as a set of changes to MIR rather than a completely separate language. The main changes are:

```text
MIR variables
    ↓
LIR registers and memory addresses
```

Assignments and operands are changed so that variable names are replaced by explicit registers or memory references. Calls are also changed: the argument list is removed because parameter passing is assumed to be implemented through a lower-level calling convention, such as registers or stack slots.

### Why LIR Is Needed

Some compiler tasks cannot be expressed well at the MIR/HIR level. For example:

```text
register allocation
instruction scheduling
addressing-mode selection
machine idiom recognition
spill/reload insertion
```

These tasks require concrete information about where values live and how operations will map to the target architecture.

At the MIR level, the compiler may say:

```text
x <- y + z
```

At the LIR level, the compiler may need to say something closer to:

```text
r1 <- r2 + r3
```

or:

```text
r1 <- [fp - 8]
[fp - 12] <- r1
```

The latter form exposes registers, stack-frame locations, and memory addresses. That is why LIR is appropriate for late optimization and backend work.

---

## Relationship Between the Three IRs

The three representations are best understood as different abstraction levels, not unrelated languages.

|IR|Level|Main Purpose|Preserves / Exposes|
|---|--:|---|---|
|HIR|High|Loop, array, and source-structure-sensitive optimization|array subscripts, structured loops, structured conditionals|
|MIR|Medium|General-purpose optimization and explanation|three-address style operations, calls, branches, temporaries|
|LIR|Low|Machine-aware optimization and code generation|registers, memory addresses, calling convention details|

A useful mental model is:

```text
HIR answers: What did the source program structurally say?

MIR answers: What computations and control flow does the program perform?

LIR answers: How will those computations map onto registers, memory, and machine-level control flow?
```

## Example: Array Access Across Levels

Consider a simple source-level operation:

```c
A[i] = A[i] + 1;
```

At the **HIR** level, the compiler wants to preserve the array access:

```text
A[i] <- A[i] + 1
```

This is useful for dependence analysis and loop transformations.

At the **MIR** level, the compiler may lower the access into address computation:

```text
t1 <- addr A
t2 <- 4 * i
t3 <- t1 + t2
t4 <- *t3
*t3 <- t4 + 1
```

This is better for general optimization because the computation is explicit.

At the **LIR** level, the compiler may express the same logic using registers and memory addresses:

```text
r1 <- addr A
r2 <- 4 * r_i
r3 <- r1 + r2
r4 <- [r3]
[r3] <- r4 + 1
```

This form is closer to machine code and therefore appropriate for backend optimization.

## Key Takeaways

### 1. MIR Is the Book’s Main Working IR

MIR is the default representation used for most examples. It is close to three-address code and supports ordinary compiler operations such as assignments, calls, branches, returns, pointer dereference, and field access.

### 2. HIR Preserves High-Level Structure

HIR keeps source-like constructs such as structured loops and array subscripts. This is important for dependence analysis, vectorization, parallelization, and cache-oriented transformations.

### 3. LIR Exposes Machine-Level Details

LIR replaces abstract variables with registers and memory addresses. It is used when optimizations need to reason about the actual target-machine representation.

### 4. The Three IRs Form a Practical Multi-Level Design

The book’s IR design reflects a common compiler engineering principle: no single representation is best for every phase. High-level analysis, general optimization, and low-level code generation each benefit from a different level of abstraction.

### 5. The Compiler May Mix Levels Temporarily

The text notes that MIR may sometimes be mixed with HIR or LIR features to make a specific point or represent an intermediate lowering stage. This is realistic: production compilers often have transitional forms during lowering rather than perfectly separated IR layers.
