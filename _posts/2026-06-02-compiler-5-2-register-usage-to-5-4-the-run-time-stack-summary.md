---
layout: single
title: "[Compiler] 5.2 Register Usage ~ 5.4 The Run-Time Stack Summary"
date: 2026-06-02
categories: Compiler
tags: [Compiler, Runtime-Support]
---

> **Source note:** This article is a study note based on Sections 5.2–5.4 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Overview

Sections **5.2–5.4** explain how a compiler supports procedure execution at run time using three core mechanisms:

```text
Registers
   ↓
Local Stack Frame
   ↓
Run-Time Stack
```

The compiler must decide:

```text
Which values stay in registers?
Which values go into the stack frame?
How does one procedure call another?
How are local and nonlocal variables accessed?
```

This part is especially important because **run-time support is not just implementation detail**. It defines how high-level language features such as local variables, recursion, nested procedures, and procedure calls are mapped onto real machine resources.

---

## 5.2 Register Usage

Registers are the fastest storage locations available to generated code. The compiler wants to use them aggressively, but registers are limited and must also serve special run-time purposes.

A register may be used for:

```text
General computation
    ├── local variables
    ├── temporaries
    ├── expression results
    └── frequently used values

Run-time management
    ├── stack pointer
    ├── frame pointer
    ├── return address
    ├── procedure arguments
    ├── return values
    ├── static link
    └── dynamic link
```

The key tradeoff is:

```text
More registers reserved for run-time support
        ↓
Fewer registers available for optimization
        ↓
More spilling may be needed

More registers available for variables
        ↓
Faster local computation
        ↓
But procedure calls and stack access may become harder
```

So register usage is both an **optimization problem** and an **ABI/runtime design problem**.

### Register Usage as a Contract

The compiler, procedure-call convention, linker, debugger, and runtime system must agree on how registers are used. For example:

|Register Kind|Typical Use|
|---|---|
|Argument registers|Pass parameters to a callee|
|Return-value registers|Return procedure results|
|Stack pointer register|Track current top of stack|
|Frame pointer register|Provide stable access to stack-frame objects|
|Callee-saved registers|Must be restored by the called procedure|
|Caller-saved registers|Must be saved by the caller if needed after call|

A compiler cannot freely allocate every hardware register to user variables because some registers have fixed roles in the procedure-call protocol.

---

## 5.3 The Local Stack Frame

A **local stack frame** is the memory area allocated for one active procedure invocation.

Even if registers are fast, not every value can stay in registers. A stack frame is needed when:

```text
Value cannot fit in available registers
Value's address is taken
Value must survive across procedure calls
Value is too large, such as an array or structure
Compiler needs spill slots
Debugger needs stable locations
Procedure needs outgoing argument space
```

A simplified stack frame looks like this:

```text
Higher memory addresses
        │
        │  Previous stack frame
        │
        ├────────────────────────
        │  Return address
        │  Dynamic link
        │  Static link
        │  Saved registers
        │  Local variables
        │  Compiler temporaries
        │  Outgoing arguments
        ├────────────────────────
        │
Lower memory addresses
```

The textbook discusses stack-frame layout through the relationship between the **stack pointer (`sp`)** and the **frame pointer (`fp`)**. Figure 5.5 illustrates a stack frame using both pointers, where `fp` provides a stable reference point and `sp` tracks the current stack position.

### Stack Pointer vs Frame Pointer

The two pointers solve different problems:

```text
sp = where the stack currently ends
fp = stable base of the current frame
```

More concretely:

|Pointer|Role|Strength|
|---|---|---|
|`sp`|Points to current stack top|Good for push/pop and outgoing calls|
|`fp`|Points to stable frame base|Good for fixed-offset access to locals|

A frame object can be accessed like this:

```text
local variable x = [fp + fixed_offset]
temporary t      = [sp + fixed_offset]
```

### Why `alloca()` Complicates Stack Access

The C function `alloca()` dynamically extends the current stack frame.

Before `alloca()`:

```text
fp ── stable frame base
sp ── original stack top
```

After `alloca()`:

```text
fp ── same stable frame base
sp ── moved downward
```

Visualization:

```text
Before alloca():

fp ────────────────┐
                   │ fixed locals
sp ────────────────┘


After alloca():

fp ────────────────┐
                   │ fixed locals
                   │ dynamically allocated area
sp ────────────────┘
```

If the compiler accessed local variables only through `sp`, then after `alloca()` the offsets would change. That is why supporting `alloca()` usually requires both `fp` and `sp`: `fp` keeps local-variable offsets stable, while `sp` tracks the changing stack top.

---

## 5.4 The Run-Time Stack

The **run-time stack** is the full sequence of active procedure frames.

Each active call gets its own frame:

```text
main() frame
    ↓
f() frame
    ↓
g() frame
    ↓
h() frame
```

If a procedure is recursive, the same procedure can have multiple active frames:

```text
fact(3)
  ↓
fact(2)
  ↓
fact(1)
```

Each invocation needs its own locals, saved registers, and return information.

The textbook emphasizes that at run time the compiler no longer has the full symbol-table structure available. Therefore, during compilation, it must assign concrete addressing information that reflects the source program’s scope rules.

### Dynamic Link

A **dynamic link** connects a frame to the caller’s frame.

```text
Current frame
    ↓ dynamic link
Caller frame
    ↓ dynamic link
Caller’s caller frame
```

The dynamic link follows the **actual call chain**.

It answers:

```text
Who called me?
Where should control return?
Which frame should be restored after return?
```

### Static Link

A **static link** connects a frame to the frame of its lexically enclosing procedure.

This matters for languages with nested procedures:

```text
procedure f()
    variable x

    procedure g()
        use x
```

When `g()` accesses `x`, it must find the active frame of `f()`.

```text
g's frame
   ↓ static link
f's frame
   ↓ offset of x
x
```

The static link follows the **lexical nesting structure**, not necessarily the runtime caller chain.

### Dynamic Link vs Static Link

|Link|Follows|Main Purpose|
|---|---|---|
|Dynamic link|Runtime call order|Return and restore caller frame|
|Static link|Lexical nesting|Access nonlocal variables|

This distinction is one of the most important ideas in Section 5.4.

Visualization:

```text
Call chain:

main → f → g → h

Dynamic links follow this chain.


Lexical nesting:

f
└── g
    └── h

Static links follow this nesting structure.
```

The two chains may look similar in simple examples, but they are conceptually different.

---

## Static-Link Setup Rules

When procedure `A` calls procedure `B`, the compiler must determine what static link should be placed in `B`’s new frame.

The textbook gives three cases.

### Case 1 — Callee is nested directly inside caller

```text
A
└── B
```

If `A` calls `B`, then:

```text
B.static_link = A.frame
```

Visualization:

```text
B frame
  ↓ static link
A frame
```

### Case 2 — Callee is at the same nesting level as caller

```text
Outer
├── A
└── B
```

If `A` calls `B`, then both procedures share the same lexical parent.

```text
B.static_link = A.static_link
```

Visualization:

```text
A frame ── static link ──► Outer frame
B frame ── static link ──► Outer frame
```

### Case 3 — Callee is higher in the nesting structure

If the callee is `n` levels higher than the caller, the compiler follows static links upward and copies the appropriate one.

```text
Caller frame
    ↓ static link
Parent frame
    ↓ static link
Grandparent frame
```

This allows the new frame to point to the correct lexical environment.

---

## Accessing Nonlocal Variables

Suppose the source program has this structure:

```text
procedure f()
    int x

    procedure g()
        procedure h()
            use x
```

When `h()` uses `x`, the generated code cannot access `x` directly from `h`’s frame, because `x` belongs to `f`.

The compiler generates code equivalent to:

```text
h frame
  ↓ static link
g frame
  ↓ static link
f frame
  ↓ offset(x)
x
```

In low-level form:

```text
r1 = [fp + static_link_offset]   // reach g frame
r2 = [r1 + static_link_offset]   // reach f frame
r3 = [r2 + x_offset]             // load x
```

So nonlocal access is:

```text
follow static links
        ↓
arrive at correct lexical frame
        ↓
use known offset inside that frame
```

This is slower than local access, but it is correct for statically nested scopes.

---

## Display Optimization

A **display** is an alternative to repeatedly following static links.

Instead of walking links one by one:

```text
current frame
  ↓
parent frame
  ↓
grandparent frame
```

the runtime keeps an array-like structure:

```text
display[0] = global frame
display[1] = frame at lexical depth 1
display[2] = frame at lexical depth 2
display[3] = frame at lexical depth 3
```

Then a nonlocal variable can be accessed more directly:

```text
frame = display[lexical_depth]
value = [frame + variable_offset]
```

Comparison:

|Method|Access Pattern|Advantage|Cost|
|---|---|---|---|
|Static link chain|Follow links repeatedly|Simple frame structure|Slower for deeply nested access|
|Display|Directly index lexical depth|Faster nonlocal access|Needs display maintenance|

A display can be stored in memory or registers. If stored in registers, it can make nonlocal access faster, but it also consumes registers that might otherwise be used for optimization.

---

## Key Takeaways

### 1. Register usage is a runtime design decision

Registers are not only for fast variables. They are also used for arguments, return values, stack management, links, and calling conventions.

### 2. A stack frame represents one procedure invocation

Every active procedure call needs storage for locals, saved registers, temporaries, parameters, links, and return information.

### 3. `sp` and `fp` serve different purposes

`sp` tracks the moving top of the stack. `fp` gives a stable base for accessing fixed frame objects.

### 4. Dynamic links and static links solve different problems

Dynamic links follow the runtime call chain. Static links follow lexical nesting and enable access to nonlocal variables.

### 5. Displays optimize nonlocal access

A display avoids repeated static-link traversal, but it introduces extra maintenance cost and may increase register pressure.
