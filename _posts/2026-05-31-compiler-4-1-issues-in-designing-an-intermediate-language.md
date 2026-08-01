---
layout: single
title: "[Compiler] 4.1 Issues in Designing an Intermediate Language"
date: 2026-05-31 18:30:33 +0800
categories: Compiler
tags: [Compiler, Intermediate-Representation]
toc: true
toc_sticky: true
---

## Core Idea

Section 4.1 explains that designing an intermediate language is not a purely mechanical or scientific process. It is largely an engineering design problem: the compiler designer must choose an IR that fits the source languages, target architecture, optimization goals, and existing compiler infrastructure. Muchnick emphasizes that there are many feasible intermediate-code structures, so the “best” representation depends on the compiler’s purpose and constraints.

## Why IR Design Matters

An intermediate representation sits between the front end and the back end of a compiler.

```text
Source Language
    ↓
Front End
    ↓
Intermediate Representation
    ↓
Optimization / Code Generation
    ↓
Target Machine Code
```

The choice of IR directly affects:

- how easy it is to implement optimizations,
    
- how expensive those optimizations are,
    
- how portable the compiler is across architectures,
    
- how much source-language information is preserved,
    
- how easily the IR can be lowered into machine code.
    

A poor IR choice may make some optimizations difficult or even impractical. For example, an IR designed around stack-based expression evaluation may not be ideal for a load-store RISC architecture with many registers.

---

## Reusing an Existing IR vs. Designing a New One

A compiler team can either reuse an existing intermediate representation or design a new one.

### Reusing an Existing IR

Reusing an IR can save implementation effort because existing compiler infrastructure, tools, and code generators may already depend on it. However, reuse only works well if the IR is appropriate for the new compiler’s source language and target architecture.

The main tradeoff is:

```text
Reuse existing IR
    → saves engineering cost
    → may require porting/adaptation
    → may limit optimization quality
```

The textbook gives `ucode` as an example. It was suitable for stack-machine-style expression evaluation, but less naturally suited to load-store machines with large register sets. Because of this mismatch, HP and MIPS compilers translated `ucode` into other forms for optimization.

### Designing a New IR

Designing a new IR gives more control, but introduces many design decisions:

- What abstraction level should the IR have?
    
- How machine-dependent should it be?
    
- What source-language constructs must it express?
    
- What optimizations should it support efficiently?
    
- How well can it be translated into target-machine code?
    

This is why IR design is a central compiler architecture decision rather than just a syntax choice.

## Major Design Dimensions

### 1. Level of Abstraction

The IR may be high-level, medium-level, or low-level.

```text
High-level IR
    preserves source-like structure

Medium-level IR
    language-independent optimization form

Low-level IR
    close to target-machine instructions
```

A high-level IR is useful when the compiler needs source-level information, such as array subscripts or loop structure. A low-level IR is useful when the compiler needs explicit registers, addresses, and machine-like operations.

### 2. Machine Dependence

An IR can be mostly machine-independent or highly target-specific.

A machine-independent IR is easier to reuse across architectures, while a machine-dependent IR can expose target details needed for low-level optimization and instruction selection.

### 3. Expressiveness

The IR must be expressive enough to represent the source languages being compiled. A compiler for C, Fortran, Ada, or object-oriented languages may need different constructs for arrays, procedures, scopes, calls, memory references, and type-related operations.

### 4. Optimization Suitability

Different optimizations prefer different IR forms.

For example:

```text
Array dependence analysis
    prefers source-like subscripts

Constant folding / strength reduction
    prefers explicit arithmetic expressions

Register allocation
    prefers explicit registers and addresses
```

This means one IR rarely serves every optimization perfectly.

---

## Using Multiple IR Levels

The section also introduces the idea of using more than one intermediate representation. Instead of forcing one IR to serve every compiler phase, a compiler may translate through several IRs.

```text
HIR
    ↓
MIR
    ↓
LIR
    ↓
Machine Code
```

This approach allows each phase to operate on the representation that best fits its task.

For example, the same C expression:

```c
a[i][j + 2]
```

can be represented differently depending on the compiler stage:

### High-Level Form

```text
a[i, j + 2]
```

This preserves the array structure and is useful for dependence analysis.

### Medium-Level Form

```text
addr(a) + 4 * (i * 20 + j + 2)
```

This makes address computation explicit and is useful for algebraic simplification or strength reduction.

### Low-Level Form

```text
load i
load j
compute offset
load from base + offset
```

This exposes concrete memory addressing and register usage, making it suitable for low-level optimization and code generation.

## Multi-Level IR Within One Representation

Another option is to design a single IR that supports multiple levels of abstraction. For example, an array reference might be representable both as:

```text
a[i, j]
```

and as:

```text
base(a) + offset(i, j)
```

The first form is better for dependence analysis, while the second is better for low-level arithmetic optimization.

This shows an important principle: the same program operation may need different representations depending on which compiler pass is analyzing it.

---

## External vs. Internal IR Representation

The textbook also distinguishes between the IR used inside the compiler and the IR printed for debugging or storage.

Internally, compilers usually store IR in compact binary structures, often using pointers to symbol-table entries. This is efficient for compiler passes.

Externally, the IR may be printed in a readable textual form for debugging. Some compilers also need a persistent external form if intermediate code is saved for later interprocedural optimization or cross-module procedure integration.

Important external representation issues include:

- how to represent pointers in a position-independent way,
    
- how to make compiler-generated temporaries unique,
    
- how to keep the representation compact and fast to read/write.
    

A compiler may therefore use two external forms:

```text
Human-readable textual IR
    → debugging and inspection

Compact binary IR
    → fast storage, loading, interprocedural optimization
```

## Key Takeaways

### 1. IR Design Is a Compiler Architecture Decision

Intermediate-language design determines what information is preserved, what optimizations are easy, and how code generation proceeds.

### 2. No Single IR Is Perfect for Every Task

High-level IR helps preserve source semantics; low-level IR exposes machine details. Medium-level IR often works well for general-purpose optimization.

### 3. Optimization Requirements Should Drive IR Design

An IR should not merely represent the program correctly. It should also make important compiler analyses and transformations efficient.

### 4. Multiple IRs Are Often Practical

Real compilers frequently translate through several IR levels because different compiler phases need different views of the program.

### 5. External IR Representation Matters

Debugging, cross-module optimization, and compiler engineering workflows may require readable or persistent IR formats, not just internal binary structures.