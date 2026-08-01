---
layout: single
title: "[Compiler] 5.6 Procedure Prologues, Epilogues, Calls, and Returns"
date: 2026-06-05 16:16:18 +0800
categories: Compiler
tags: [Compiler, Runtime-Support]
toc: true
toc_sticky: true
---

> **Source note:** This article is a study note based on Section 5.6 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Section 5.6 explains how procedure invocation is implemented at run time. A procedure call is not just a `jump`. It is a structured **caller–callee handshake** that transfers:

- control,
    
- argument values,
    
- register ownership,
    
- stack-frame responsibility,
    
- return values.
    

The textbook defines procedure execution as five major phases: the caller prepares the call, the callee prologue builds its environment, the procedure body runs, the callee epilogue restores the environment, and the caller receives the returned value.

```text
Caller
  ├─ evaluate arguments
  ├─ place arguments in registers/stack
  ├─ save caller-saved registers
  ├─ compute static link if needed
  └─ branch to callee

Callee
  ├─ prologue: build stack frame
  ├─ body: execute procedure code
  └─ epilogue: restore state and return

Caller
  ├─ restore caller-saved registers
  └─ use returned value
```

---

## The Procedure Call Sequence

The **procedure call** is generated in the caller. Its job is to prepare everything the callee needs before control is transferred.

The caller typically performs these steps:

1. Evaluate each argument.
    
2. Put each argument in the required register or stack slot.
    
3. Determine the callee’s code address.
    
4. Save registers that the caller is responsible for preserving.
    
5. Compute the static link if the language supports nested procedures.
    
6. Save the return address and branch to the callee.
    

### Visualization — Caller to Callee Transfer

```text
Before call

Caller frame
+-------------------------+
| caller local variables  |
| live temporary values   |
| caller-saved registers  |
+-------------------------+
            |
            |  call f(...)
            v

During transfer

Argument registers / stack slots
+-------------------------+
| arg1                    |
| arg2                    |
| arg3                    |
| static link, if needed  |
| return address          |
+-------------------------+
            |
            v

Callee entry
+-------------------------+
| prologue starts here    |
+-------------------------+
```

The important compiler-design point is that the caller and callee must agree on the exact convention. If the caller places argument `x` in register `r1`, the callee must generate code that reads `x` from `r1`.

---

## Procedure Prologue

The **prologue** is code automatically inserted at the beginning of the callee. It establishes the callee’s runtime environment.

Typical prologue work:

- save the old frame pointer,
    
- make the old stack pointer become the new frame pointer,
    
- allocate the callee’s stack frame,
    
- save callee-saved registers that the callee will use,
    
- construct a display if the runtime model uses displays.
    

### Visualization — Prologue Builds a New Frame

```text
Before prologue

Caller frame
+-------------------------+
| caller locals           |
| caller temporaries      |
+-------------------------+
            ^
            |
          old fp


After prologue

Callee frame
+-------------------------+
| saved old frame pointer |
| static link             |
| return address          |
| callee-saved registers  |
| local variables         |
| temporaries             |
+-------------------------+
            ^
            |
          new fp

Caller frame
+-------------------------+
| caller locals           |
| caller temporaries      |
+-------------------------+
```

The prologue is what makes local-variable access and stack-frame-relative addressing possible inside the procedure.

---

## Procedure Epilogue and Return

The **epilogue** is the mirror image of the prologue. It runs when the callee is ready to return.

Typical epilogue work:

- restore callee-saved registers,
    
- place the return value in the agreed return location,
    
- recover the old stack pointer and frame pointer,
    
- branch to the saved return address.
    

```text
Callee epilogue
    ↓
restore callee-saved registers
    ↓
put return value in return register / memory
    ↓
restore old sp and fp
    ↓
jump to return address
    ↓
caller continues after call
```

The caller then restores caller-saved registers and uses the returned value. This separation is important because some registers are the caller’s responsibility, while others are the callee’s responsibility.

---

## Register Preservation: Caller-Saved vs Callee-Saved

A central issue in procedure calls is deciding **who saves which registers**.

If every caller saved every register, calls would be expensive. If every callee saved every register, calls would also be expensive. So calling conventions divide registers into categories.

The textbook describes four major register classes:

|Register class|Meaning|
|---|---|
|Dedicated registers|Used only for special calling-convention roles, such as stack pointer or frame pointer|
|Caller-saved registers|Saved by the caller if their values are needed after the call|
|Callee-saved registers|Saved by the callee if the callee wants to use them|
|Scratch registers|Not preserved across calls|

This division is architecture-dependent. For example, SPARC register windows, Intel 386’s small heterogeneous register set, and architectures that share integer/floating-point register files all affect the optimal partition.

### Visualization — Register Responsibility

```text
Register classes across a call

Dedicated
  sp, fp, return-address register
  → controlled by calling convention

Caller-saved
  caller needs value after call?
      yes → caller saves before call
      no  → no save needed

Callee-saved
  callee wants to use register?
      yes → callee saves in prologue and restores in epilogue
      no  → no save needed

Scratch
  temporary only
  → no preservation guarantee
```

The engineering goal is to avoid unnecessary memory traffic while preserving correctness.

---

## Parameter Passing in Registers: Flat Register File

On architectures with many general-purpose registers, parameters are commonly passed in registers.

A typical convention designates:

- several integer registers for integer arguments,
    
- several floating-point registers for floating-point arguments,
    
- stack locations for extra arguments.
    

For a call like:

```c
f(i, x, j)
```

where `i` and `j` are integers and `x` is a single-precision floating-point value, the textbook’s model passes `i` and `j` in the first two integer parameter registers and `x` in the first floating-point parameter register.

```text
f(i, x, j)

Integer parameter registers
+------+---------+
| r1   | i       |
| r2   | j       |
+------+---------+

Floating-point parameter registers
+------+---------+
| f0   | x       |
+------+---------+
```

If there are too many arguments to fit in registers, the remaining arguments are passed on the stack. Large value parameters may be passed by address, with the callee copying the value into its own frame if necessary.

---

## Return Values

Return values are usually handled similarly to arguments.

Common convention:

```text
small integer return value        → integer return register
small floating-point return value → floating-point return register
large structure/union return      → caller-provided memory
```

For large return values, the caller provides storage and passes a hidden pointer to that storage. The callee writes the result there. This avoids returning a pointer to callee-owned storage, which would disappear after return, and helps support reentrant procedures.

```text
Large return value

Caller
  allocates result storage
      ↓
  passes hidden pointer
      ↓

Callee
  writes result into caller-provided storage
      ↓

Caller
  reads result after return
```

---

## Parameter Passing on the Run-Time Stack

Some architectures pass parameters mainly through the run-time stack. This is common for machines with few registers or convenient stack-manipulation instructions.

The textbook gives Intel 386-style `pushl` as an example. Instead of separately storing an argument and adjusting the stack pointer, a single push instruction can do both.

```asm
pushl 5      ; push third argument onto stack
```

### Stack-Based Argument Layout

```text
Higher addresses
+-------------------+
| 3rd argument      |
| 2nd argument      |
| 1st argument      |
| static link       |
| return address    |
| caller's ebp      | ← frame pointer
| local variables   |
| saved registers   |
+-------------------+
Lower addresses
```

In this model, the callee accesses arguments using offsets from the frame pointer. The model is simple and ABI-friendly, but it usually creates more memory traffic than register-based passing.

---

## Register Windows

SPARC-style **register windows** reduce call overhead by making the caller’s output registers overlap with the callee’s input registers.

The textbook notes that register windows can reduce loads and stores because arguments and return values can often remain in registers rather than being saved to memory. Typical SPARC implementations provide multiple windows, giving the effect of a much larger register file while keeping instruction register fields small.

### Visualization — Overlapping Register Windows

```text
Caller window

+-------------------+
| caller locals     |
+-------------------+
| caller outs o0-o5 |  ─────┐
+-------------------+       │
                            │ overlap
                            v
Callee window

+-------------------+
| callee ins i0-i5  |
+-------------------+
| callee locals     |
+-------------------+
| callee outs o0-o5 |
+-------------------+
```

The caller places arguments in its `out` registers. After the call, those same physical registers are viewed as the callee’s `in` registers.

Extra arguments still go on the stack, so even register-window architectures need a stack-based fallback.

---

## Procedure-Valued Variables

Some languages allow procedures to be stored in variables and called indirectly.

For example:

```text
p := some_procedure
call p(...)
```

This is harder than a normal direct call because the compiler may not know the target procedure at compile time. If nested procedures are allowed, the call also needs the correct **static link**.

The textbook’s solution is a **procedure descriptor**. Instead of storing only the procedure’s code address, the variable points to a descriptor containing both the procedure address and its static link.

```text
Procedure descriptor

+----------------------+
| procedure address    |
+----------------------+
| static link          |
+----------------------+
```

Calling through the descriptor requires:

```text
load procedure address
    ↓
load static link
    ↓
perform indirect call
```

This preserves the callee’s lexical environment even when the procedure is called through a variable.

---

## Why This Section Matters

This section matters because procedure calls occur constantly in real programs. A poor calling convention creates unnecessary saves, restores, loads, stores, and stack traffic.

The compiler must balance:

- correctness of argument passing,
    
- efficient register usage,
    
- stack-frame layout,
    
- static-link support,
    
- direct and indirect calls,
    
- return-value handling,
    
- architecture-specific mechanisms.
    

```text
Good calling convention
    ↓
less save/restore traffic
    ↓
faster procedure calls
    ↓
better whole-program performance
```

Procedure-call implementation is also foundational for later optimizations, especially leaf-routine optimization, shrink wrapping, inlining, tail-call optimization, and interprocedural register allocation.

---

## Key Takeaways

### 1. A procedure call is a protocol

Caller and callee must agree on where arguments, return values, saved registers, frame pointers, and static links live.

### 2. Prologues and epilogues manage stack-frame lifetime

The prologue creates the callee’s environment. The epilogue restores the caller’s environment.

### 3. Register-saving policy is performance-critical

Caller-saved and callee-saved conventions reduce unnecessary preservation work, but the optimal split depends on the architecture.

### 4. Register passing is fast but incomplete

Register arguments are efficient, but extra arguments and large values still require stack or memory-based conventions.

### 5. Register windows reduce memory traffic

SPARC-style register windows turn caller output registers into callee input registers, reducing explicit argument movement.

### 6. Procedure-valued variables need more than a code address

When nested procedures and indirect calls are allowed, procedure descriptors must carry both the procedure address and the static link.