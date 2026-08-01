---
layout: single
title: "[Compiler] 12.1 Constant-Expression Evaluation, or Constant Folding"
date: 2026-05-21
categories: Compiler
tags: [Compiler, Optimization]
---

> **Source note:** This article is a study note based on Section 12.1 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Constant-expression evaluation, usually called **constant folding**, is an early compiler optimization that evaluates expressions at compile time when their operands are already known constants.

Instead of generating runtime code for an expression such as:

```c
x = 3 + 4;
````

the compiler evaluates `3 + 4` during compilation and rewrites the code as:

```c
x = 7;
```

The basic idea is simple:

```text
constant operands
        ↓
compile-time evaluation
        ↓
replace expression with constant result
```

This reduces runtime work and often exposes new opportunities for later optimizations such as constant propagation, dead-code elimination, and branch simplification.

## Basic Transformation

### Before Optimization

```c
int x;
x = 10 * 20;
```

The expression `10 * 20` has no runtime dependency. Both operands are constants.

### After Constant Folding

```c
int x;
x = 200;
```

The generated code no longer needs to emit a multiplication instruction for this expression.

In intermediate representation, the transformation can be viewed as:

```text
t1 = 10 * 20
    ↓
t1 = 200
```

The compiler replaces an operator expression with a literal constant result.

## Where Constant Folding Fits in the Optimizer

Constant folding belongs to the group of **early optimizations**. Its basic form does not require full data-flow analysis, because the compiler only needs to inspect whether the operands of a particular expression are constant.

However, its effectiveness increases significantly when combined with data-flow-based optimizations.

```text
Constant Propagation
        ↓
more operands become known constants
        ↓
Constant Folding
        ↓
simpler expressions
        ↓
Dead-Code Elimination / Branch Simplification
```

For example:

```c
int a = 4;
int b = a + 6;
```

Constant propagation can first discover that `a` is always `4`, after which constant folding can rewrite:

```c
int b = 4 + 6;
```

into:

```c
int b = 10;
```

So constant folding is not just a standalone local cleanup. It is also a reusable helper transformation that other optimization passes can invoke whenever they expose new constant expressions.

## Applicability by Data Type

Constant folding is safe only when the compile-time result is guaranteed to match the target machine’s runtime behavior. Different kinds of constants have different correctness risks.

### Boolean Expressions

Boolean constant folding is generally straightforward.

```c
if (true && false) {
    foo();
}
```

can become:

```c
if (false) {
    foo();
}
```

Then a later control-flow or dead-code elimination pass may remove the unreachable branch.

```text
true && false
      ↓
false
      ↓
unreachable branch may be removed
```

Boolean operations usually do not raise hardware-level arithmetic exceptions, so this case is the cleanest.

### Integer Expressions

Integer constant folding is also usually valid, but there are important exceptions.

Safe example:

```c
x = 8 + 9;
```

becomes:

```c
x = 17;
```

Problematic example:

```c
x = 1 / 0;
```

This expression would raise a runtime exception if executed. A compiler cannot blindly replace it with an arbitrary value. It must preserve the source language’s semantics.

Another subtle case is overflow:

```c
x = INT_MAX + 1;
```

Whether this can be folded depends on the language and target semantics. If the language requires overflow detection, folding the expression at compile time must still preserve the same observable behavior as runtime execution.

### Addressing Arithmetic

Addressing arithmetic is a special case where constant folding is generally safe and useful.

For example:

```text
base + 16
```

or:

```text
frame_pointer + constant_offset
```

can often be folded into an addressing mode or a simpler address computation.

The key point is that address arithmetic is usually part of compiler-generated address calculation, and overflow behavior in this context does not carry the same source-level semantic meaning as ordinary integer arithmetic.

### Floating-Point Expressions

Floating-point constant folding is more dangerous than integer folding.

A compiler must ensure that compile-time floating-point evaluation behaves exactly like the target processor’s runtime floating-point behavior.

Potential issues include:

- rounding mode,
    
- infinities,
    
- NaNs,
    
- denormalized values,
    
- floating-point exceptions,
    
- differences between host and target floating-point formats.
    

For example:

```c
double x = 0.1 + 0.2;
```

The compiler must not fold this expression using a different arithmetic model from the target machine. Otherwise, the compiled program may produce a result that differs from runtime evaluation.

This is why floating-point constant folding often needs either target-accurate evaluation or a conservative policy.

## Algorithmic View

At a high level, the constant folding routine checks the expression kind and determines whether all operands are constants.

```text
Instruction
    ↓
Is it a binary expression?
    ↓
Are both operands constant?
    ↓
Evaluate binary operator at compile time
    ↓
Return simplified instruction
```

For unary expressions, the structure is similar:

```text
Instruction
    ↓
Is it a unary expression?
    ↓
Is the operand constant?
    ↓
Evaluate unary operator at compile time
    ↓
Return simplified instruction
```

Conceptually, the compiler performs this transformation:

```text
op const1 const2
        ↓
const_result
```

or:

```text
op const
    ↓
const_result
```

The important implementation detail is that the helper used to evaluate the operation must model the **target machine**, not merely the compiler host machine.

## Example — Folding a Conditional

### Before Optimization

```c
if (3 * 4 == 12) {
    fast_path();
} else {
    slow_path();
}
```

First, the arithmetic expression is folded:

```c
if (12 == 12) {
    fast_path();
} else {
    slow_path();
}
```

Then the comparison is folded:

```c
if (true) {
    fast_path();
} else {
    slow_path();
}
```

A later control-flow cleanup pass can simplify this into:

```c
fast_path();
```

### Optimization Chain

```text
Arithmetic constant folding
        ↓
Boolean constant folding
        ↓
Branch simplification
        ↓
Dead-code elimination
```

This example shows why constant folding is often more valuable as part of a pipeline than as an isolated peephole transformation.

## Relationship to Constant Propagation

Constant folding and constant propagation are closely related but not the same.

Constant propagation discovers that a variable has a known constant value:

```c
a = 5;
b = a + 3;
```

After propagation:

```c
b = 5 + 3;
```

Constant folding then evaluates the expression:

```c
b = 8;
```

So the two optimizations form a natural pair:

```text
Constant propagation answers:
“What values are known constants?”

Constant folding answers:
“What expressions can now be evaluated?”
```

In practice, many optimizer pipelines repeatedly expose and fold constants as other passes simplify the program.

## Why Constant Folding Should Be a Reusable Subroutine

Constant folding is best implemented as a helper routine that can be invoked whenever needed.

This design makes sense because many passes can create new constant expressions:

```text
Inlining
    ↓
Constant propagation
    ↓
Algebraic simplification
    ↓
Value numbering
    ↓
SCCP
```

Any of these may expose expressions such as:

```c
2 * 8
x + 0
true && false
```

Rather than implementing constant evaluation logic separately inside every pass, the optimizer should centralize it as a reusable routine.

This also improves correctness because tricky cases such as overflow, division by zero, floating-point exceptions, and target-specific behavior are handled in one place.

## Engineering Implications

Constant folding looks simple, but a production-quality implementation must be careful about semantic equivalence.

A correct implementation should consider:

- whether the operation can raise an exception,
    
- whether the source language defines overflow behavior,
    
- whether compile-time evaluation matches target-machine behavior,
    
- whether floating-point exceptions and special values are observable,
    
- whether folding changes when an error would be reported.
    

The guiding rule is:

```text
Fold only when the folded result has the same observable behavior
as executing the original expression at runtime.
```

This is especially important in cross-compilers, where the compiler host and target machine may have different arithmetic behavior.

## Key Takeaways

### 1. Constant Folding Evaluates Known Expressions Early

If all operands of an expression are constants, the compiler can often evaluate the expression at compile time and replace it with the result.

### 2. It Is Simple but Semantics-Sensitive

Boolean folding is usually safe, integer folding must respect exceptions and overflow semantics, and floating-point folding requires target-accurate behavior.

### 3. It Works Best with Other Optimizations

Constant propagation, inlining, SCCP, and algebraic simplification can expose new constant expressions that constant folding can then simplify.

### 4. It Should Be Implemented as a Shared Optimizer Utility

Because many passes need constant evaluation, constant folding should be structured as a reusable subroutine rather than duplicated inside individual passes.

### 5. Correctness Depends on Target Behavior

The folded result must behave as if the original operation had executed on the target machine at runtime.
