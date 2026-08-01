---
layout: single
title: "[Compiler] 12.2 Scalar Replacement of Aggregates"
date: 2026-05-21 14:25:30 +0800
categories: Compiler
tags: [Compiler, Optimization]
toc: true
toc_sticky: true
---

> **Source note:** This article is a study note based on Section 12.2 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

**Scalar Replacement of Aggregates** is an early compiler optimization that breaks an aggregate object into separate scalar temporaries.

An aggregate is a compound data object such as:

- a C `struct`,
- a Pascal `record`,
- a complex number represented as `{ real, imag }`,
- or any object made of multiple fields.

Instead of treating the whole object as one memory-based structure, the compiler replaces selected fields with independent scalar variables.

```c
snack.variety = APPLE;
snack.shape = ROUND;
````

can be transformed conceptually into:

```c
t1 = APPLE;   // represents snack.variety
t2 = ROUND;   // represents snack.shape
```

The main goal is not simply to remove structure accesses. The real goal is to make aggregate components available to scalar optimizations such as constant propagation, copy propagation, register allocation, and dead-code elimination.

---

## Why This Optimization Matters

Aggregate fields are often stored and accessed through memory. Because memory accesses may be affected by aliasing, the compiler must usually be conservative.

For example:

```c
snack.variety = APPLE;
foo(&snack);
x = snack.variety;
```

The call `foo(&snack)` may modify `snack.variety`, so the compiler cannot safely assume that `x` is still `APPLE`.

Scalar replacement becomes possible when the compiler can prove that:

1. the aggregate component has a simple scalar value,
    
2. the component is not aliased,
    
3. the whole aggregate object is not aliased in a way that could change the field unexpectedly.
    

Once these conditions are satisfied, the field can be treated like an ordinary scalar variable.

```text
aggregate field access
        ↓
scalar temporary
        ↓
scalar optimization opportunity
```

This is why scalar replacement is especially useful early in the optimization pipeline.

---

## Basic Transformation

The optimization divides a structure into distinct scalar variables.

For example, given:

```c
typedef struct fruit {
    VARIETY variety;
    SHAPE shape;
} FRUIT;

FRUIT snack;
```

the compiler can conceptually create:

```c
VARIETY t1;   // snack.variety
SHAPE   t2;   // snack.shape
```

Then assignments to the aggregate fields are rewritten:

```c
snack.variety = APPLE;
snack.shape = ROUND;
```

into:

```c
t1 = APPLE;
t2 = ROUND;
```

After this transformation, later passes can reason about `t1` and `t2` as normal scalar values.

---

## Example — Fruit Structure

Muchnick’s example uses a `FRUIT` structure with two fields: `variety` and `shape`.

### Before Optimization

```c
typedef enum { APPLE, BANANA, ORANGE } VARIETY;
typedef enum { LONG, ROUND } SHAPE;

typedef struct fruit {
    VARIETY variety;
    SHAPE shape;
} FRUIT;

char* Red = "red";
char* Yellow = "yellow";
char* Orange = "orange";

char*
color(FRUIT *CurrentFruit) {
    switch (CurrentFruit->variety) {
        case APPLE:
            return Red;
        case BANANA:
            return Yellow;
        case ORANGE:
            return Orange;
    }
}

main() {
    FRUIT snack;

    snack.variety = APPLE;
    snack.shape = ROUND;

    printf("%s\n", color(&snack));
}
```

At this point, the value used in the `switch` statement comes from a structure field:

```c
CurrentFruit->variety
```

This makes the value harder for the compiler to track directly.

### After Procedure Integration and Scalar Replacement

If the compiler first integrates `color()` into `main()` and then performs scalar replacement, the structure fields can be represented by temporaries:

```c
main() {
    VARIETY t1;
    SHAPE t2;
    char* t3;

    t1 = APPLE;
    t2 = ROUND;

    switch (t1) {
        case APPLE:
            t3 = Red;
            break;
        case BANANA:
            t3 = Yellow;
            break;
        case ORANGE:
            t3 = Orange;
            break;
    }

    printf("%s\n", t3);
}
```

The important change is:

```c
switch (CurrentFruit->variety)
```

becomes:

```c
switch (t1)
```

Since `t1` is assigned the constant value `APPLE`, constant propagation can simplify the switch.

### After Constant Propagation and Dead-Code Elimination

```c
main() {
    printf("%s\n", "red");
}
```

The `BANANA` and `ORANGE` cases become unreachable and are removed by dead-code elimination.

---

## Optimization Pipeline

Scalar replacement becomes powerful when combined with other optimizations.

```text
Procedure Integration / Inlining
        ↓
Scalar Replacement of Aggregates
        ↓
Constant Propagation
        ↓
Dead-Code Elimination
        ↓
Simplified Program
```

In the example:

```text
snack.variety = APPLE
        ↓
t1 = APPLE
        ↓
switch (t1)
        ↓
switch (APPLE)
        ↓
only APPLE case remains
        ↓
printf("%s\n", "red")
```

This shows why scalar replacement is an enabling optimization. It exposes information that later passes can exploit.

---

## Safety Conditions

Scalar replacement is only valid when it preserves program semantics.

### The component must be scalar

The field should contain a simple scalar value, such as:

```c
int
float
enum
pointer
```

For example:

```c
snack.variety = APPLE;
```

is a good candidate because `snack.variety` is an enum value.

### The aggregate must not be dangerously aliased

If another pointer can modify the same object, scalar replacement may be unsafe.

```c
FRUIT snack;
FRUIT *p = &snack;

snack.variety = APPLE;
p->variety = BANANA;
```

Here, replacing `snack.variety` with a temporary may hide the fact that `p->variety` modifies the same field.

### The replacement region must be clear

Scalar replacement can be applied:

- across a whole procedure,
    
- or inside a smaller region such as a loop.
    

Whole-procedure replacement gives a larger optimization scope, but it requires stronger alias guarantees. Loop-local replacement may be possible even when whole-procedure replacement is not.

---

## Procedure-Level vs Loop-Level Scalar Replacement

## Procedure-Level Replacement

In procedure-level scalar replacement, the compiler replaces aggregate fields throughout the whole procedure.

```text
whole procedure:
    snack.variety → t1
    snack.shape   → t2
```

This is usually appropriate when the compiler can prove that the aggregate object and its fields are not aliased across the procedure.

The advantage is that more code becomes available for scalar optimization.

The disadvantage is that the aliasing proof is harder.

## Loop-Level Replacement

In loop-level scalar replacement, the compiler replaces aggregate or memory-based values only inside a loop.

For example:

```c
for (...) {
    sum += z.real;
    diff += z.imag;
}
```

can be transformed conceptually into:

```c
t_real = z.real;
t_imag = z.imag;

for (...) {
    sum += t_real;
    diff += t_imag;
}
```

If the scalar values are modified inside the loop, the compiler may need to write them back after the loop.

```text
before loop:
    load aggregate fields into temporaries

inside loop:
    use scalar temporaries

after loop:
    store updated temporaries back if needed
```

Loop-level replacement can be profitable because loops execute repeatedly, so removing repeated memory accesses can have a significant impact.

---

## Why It Enables Other Optimizations

Scalar replacement is useful because it turns memory-like aggregate accesses into scalar values.

### Register Allocation

Aggregate fields stored in memory may require repeated loads and stores.

```c
x = snack.variety;
```

After scalar replacement:

```c
x = t1;
```

Now `t1` can be allocated to a register.

### Constant Propagation

```c
t1 = APPLE;
switch (t1) {
    ...
}
```

Since `t1` has a known constant value, the compiler can simplify the control flow.

### Copy Propagation

```c
t4 = t1;
```

If `t4` is only a copy of `t1`, copy propagation can replace uses of `t4` with `t1`.

### Dead-Code Elimination

After constant propagation, some branches or assignments may become unnecessary.

```text
constant propagation
        ↓
unreachable branch discovered
        ↓
dead-code elimination
```

In the fruit example, only the `APPLE` case remains, so the other switch cases become dead code.

---

## Practical Importance

This optimization is especially useful for programs that operate on complex numbers.

A complex number is often represented as a record or structure containing two real numbers:

```c
typedef struct {
    double real;
    double imag;
} Complex;
```

Without scalar replacement, repeated operations may access:

```c
z.real
z.imag
```

through memory.

With scalar replacement, the compiler can use:

```c
z_real
z_imag
```

as scalar temporaries.

```text
z.real → z_real
z.imag → z_imag
```

This improves the chance that the values stay in registers and become available to scalar optimizations.

Muchnick notes that in one double-precision complex FFT kernel from the SPEC benchmark `nasa7`, adding scalar replacement to the Sun SPARC compiler optimizations produced an additional 15% reduction in execution time.

---

## Relation to Modern Compiler Terminology

In modern LLVM terminology, this optimization is commonly associated with **SROA**:

```text
SROA = Scalar Replacement of Aggregates
```

LLVM SROA breaks aggregate memory objects, especially `alloca`-based objects, into smaller scalar pieces when it is safe to do so.

Conceptually:

```text
aggregate allocation
        ↓
field-wise scalarization
        ↓
promotion to SSA values
        ↓
scalar optimizations
```

This is closely related to the idea described by Muchnick. Aggregates block many scalar optimizations because they appear as memory objects. Once they are split into scalar values, the optimizer can reason about them more precisely.

---

## Key Takeaways

### 1. Scalar replacement breaks aggregates into scalar temporaries

It transforms structure or record fields into ordinary scalar variables when doing so is safe.

### 2. The main requirement is alias safety

The compiler must prove that neither the aggregate nor the selected fields can be unexpectedly modified through another access path.

### 3. The optimization is most useful as an enabler

Scalar replacement is valuable because it exposes opportunities for:

- register allocation,
    
- constant propagation,
    
- copy propagation,
    
- dead-code elimination.
    

### 4. It can be applied at different scopes

The compiler may apply it across a whole procedure or only within smaller regions such as loops.

### 5. It is especially useful for complex-number-heavy programs

Complex numbers are often represented as aggregates containing real and imaginary fields, making them natural candidates for scalar replacement.