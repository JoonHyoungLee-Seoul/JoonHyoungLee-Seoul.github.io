---
layout: single
title: "[Compiler] 4.2–4.5 Intermediate-Language Levels Summary"
date: 2026-05-31 18:57:54 +0800
categories: Compiler
tags: [Compiler, Intermediate-Representation]
toc: true
toc_sticky: true
---

> **Source note:** This article is a study note based on Sections 4.2–4.5 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Sections 4.2–4.5 classify intermediate representations by **abstraction level**: high-level, medium-level, low-level, and multi-level IRs. The main point is that no single representation is ideal for every compiler phase. Higher-level IRs preserve source-language structure, lower-level IRs expose machine details, and multi-level IRs try to support several views within one compiler infrastructure.

```text
Source-like structure
        ↓
High-Level IR
        ↓
Medium-Level IR
        ↓
Low-Level IR
        ↓
Machine code
```

## 4.2 High-Level Intermediate Languages

A **high-level intermediate language** stays close to the source program. It preserves structures such as loops, array subscripts, and source-level operations instead of immediately lowering everything into pointer arithmetic or machine-like instructions.

This kind of IR is especially useful when the compiler needs to reason about the program at a semantic level.

For example, an array access such as:

```c
a[i][j + 2]
```

can remain represented as a true multidimensional array reference:

```text
a[i, j + 2]
```

This is valuable because the compiler can still see:

- the array object,
    
- each subscript separately,
    
- the loop/index structure,
    
- the relationship between array accesses.
    

That makes high-level IR particularly useful for **dependence analysis** and memory-hierarchy optimizations, where preserving array structure is more informative than immediately converting the access into linear address arithmetic. The textbook explicitly notes that list-style subscript representation is useful for dependence analysis and optimizations based on it.

### Why High-Level IR Helps

```text
High-Level IR
    → keeps array subscripts visible
    → keeps loop structure visible
    → supports dependence analysis
    → supports data-cache / memory optimizations
```

The drawback is that high-level IR is usually too abstract for final code generation. Eventually, array accesses, procedure calls, and structured control flow must be lowered into explicit address computations, branches, loads, and stores.

## 4.3 Medium-Level Intermediate Languages

A **medium-level intermediate language** is a compromise between source-level structure and machine-level detail. It is less tied to the original source syntax than a high-level IR, but still not fully machine-specific.

In the textbook’s example, the same array access can be lowered into a more explicit address computation:

```text
addr(a) + offset computation
```

For the C declaration:

```c
float a[20][10];
```

an access like:

```c
a[i][j + 2]
```

can be represented in a medium-level form where the compiler explicitly computes the linearized memory offset. This form is better suited to general scalar optimizations because the computation is now expressed as arithmetic operations.

Medium-level IR is useful for optimizations such as:

- constant folding,
    
- strength reduction,
    
- loop-invariant code motion,
    
- common-subexpression elimination,
    
- copy propagation,
    
- value numbering.
    

The key advantage is that arithmetic and memory-address expressions become explicit enough for optimization, while the IR can still remain relatively machine-independent.

```text
High-Level:
    a[i, j + 2]

Medium-Level:
    *(addr(a) + computed_offset)
```

The textbook explains that the linearized form of array references is appropriate for optimizations such as constant folding, strength reduction, loop-invariant code motion, and other basic optimizations.

---

## 4.4 Low-Level Intermediate Languages

A **low-level intermediate language** is close to the target machine. It exposes details such as registers, explicit loads and stores, branches, address modes, and instruction-like operations.

This level is important because some optimizations cannot be performed well until machine-level structure is visible.

Examples include:

- instruction selection,
    
- register allocation,
    
- instruction scheduling,
    
- branch optimization,
    
- machine idiom recognition,
    
- address-mode selection.
    

At this stage, the compiler is no longer mainly asking:

```text
What does this source construct mean?
```

It is asking:

```text
How should this computation be mapped efficiently onto the target machine?
```

The textbook’s PA-RISC example illustrates this point. A MIR fragment can be translated into alternative PA-RISC code sequences depending on whether the compiler chooses to combine a load with an address update or keep the address update separate for loop-control purposes.

### Example — Load With Address Update

Some architectures provide instructions that can load from memory and update the address register at the same time.

```text
load from address
update address
```

can sometimes become:

```text
load-with-modify
```

But this is not always profitable. If the updated address is also needed for loop termination, the compiler may prefer to keep address update and loop control separate. The textbook uses this to show why low-level IR must expose enough target-machine detail for code generation choices.

## 4.5 Multi-Level Intermediate Languages

A **multi-level intermediate language** supports more than one abstraction level inside the same IR system. Instead of maintaining completely separate high-level, medium-level, and low-level IRs, a compiler may allow certain constructs to have both high-level and lowered forms.

The textbook gives Sun IR as an example: it can represent multiply subscripted array references directly, while also allowing those subscripts to be linearized into a single memory offset.

```text
Same operation, two possible IR views:

High-level view:
    a[i, j + 2]

Lower-level view:
    base_address(a) + linearized_offset
```

This is useful because different passes want different information.

```text
Dependence analysis
    prefers high-level subscript lists

Scalar optimization
    prefers explicit arithmetic

Code generation
    prefers low-level address forms
```

A multi-level IR therefore tries to avoid forcing the compiler to choose only one representation too early. It can preserve semantic information for high-level analyses while still enabling low-level transformations when needed.

---

## Comparison of IR Levels

|IR Level|Main Characteristic|Best For|Main Limitation|
|---|---|---|---|
|High-Level IR|Close to source language|Dependence analysis, loop/array reasoning, memory optimization|Too abstract for final code generation|
|Medium-Level IR|Machine-independent but explicit|General scalar optimizations|May lose some source-level structure|
|Low-Level IR|Close to target machine|Register allocation, scheduling, instruction selection|Less portable and less source-aware|
|Multi-Level IR|Combines multiple abstraction views|Flexible optimization pipeline|More complex IR design and maintenance|

## Key Takeaways

### 1. IR Level Determines Optimization Visibility

High-level IR exposes source structure; low-level IR exposes machine structure. Different optimizations need different visibility.

### 2. Array References Are the Central Example

The same array access can be represented as a source-like subscript expression, a linearized address computation, or a machine-level load sequence. Each representation supports different compiler tasks.

### 3. Medium-Level IR Is the General Optimization Workhorse

Medium-level IR is often the most useful form for machine-independent scalar optimizations because it makes computations explicit without fully committing to target-machine details.

### 4. Low-Level IR Is Necessary for Machine-Aware Optimization

Target-specific instruction choices, addressing modes, scheduling, and register allocation require a representation close to real machine code.

### 5. Multi-Level IR Reduces Premature Information Loss

A multi-level IR allows the compiler to keep high-level semantic information while also supporting lower-level transformations. This makes it powerful, but also harder to design cleanly.