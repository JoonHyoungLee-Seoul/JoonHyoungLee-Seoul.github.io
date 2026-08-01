---
layout: single
title: "[Compiler] 5.5 Parameter-Passing Disciplines"
date: 2026-06-04 15:20:55 +0800
categories: Compiler
tags: [Compiler, Runtime-Support]
toc: true
toc_sticky: true
---

## Core Idea

Section 5.5 explains how programming languages pass arguments into procedures and how results are returned from procedures at run time.

The textbook introduces five major parameter-passing disciplines:

1. **Call by value**
    
2. **Call by result**
    
3. **Call by value-result**
    
4. **Call by reference**
    
5. **Call by name**
    

It also distinguishes between:

- **arguments / actual arguments**: values or variables supplied by the caller
    
- **parameters / formal parameters**: variables inside the called procedure that receive or represent those arguments
    

In compiler implementation, parameter passing is not just a language-level feature. It affects:

- stack-frame layout,
    
- register usage,
    
- memory traffic,
    
- aliasing,
    
- procedure-call overhead,
    
- optimization opportunities.
    

---

## Call by Value

**Call by value** means the caller passes the value of an argument to the callee.

Conceptually:

```text
caller variable/value
        ↓ copied
callee formal parameter
```

Inside the callee, modifying the formal parameter does **not** modify the caller’s original variable.

Example idea:

```c
void f(int x) {
    x = 10;
}

int a = 3;
f(a);
// a is still 3
```

This is simple and efficient when the argument fits in a register, such as an integer or pointer. However, it can be expensive for large data structures like arrays because copying the whole object may require large memory traffic.

The textbook notes one optimization: if the compiler can prove that an array passed by value is not modified, it may pass the array’s address instead of copying the whole array.

Languages mentioned:

- C
    
- C++
    
- Algol 60
    
- Algol 68
    
- Ada `in` parameters, with the restriction that they are read-only inside the callee
    

---

## Call by Result

**Call by result** is the opposite direction of call by value.

Instead of passing a value from caller to callee at procedure entry, the callee produces a value and copies it back to the caller at procedure return.

Conceptually:

```text
callee formal parameter
        ↓ copied back on return
caller actual argument
```

On entry:

```text
no initial value is copied into the callee
```

On return:

```text
callee's final parameter value is copied to the caller's variable
```

This mechanism is useful for output-only parameters.

Example idea:

```text
procedure f(out x)
    x := 10
end

f(a)
// after return, a becomes 10
```

The textbook associates this mechanism with Ada `out` parameters.

---

## Call by Value-Result

**Call by value-result** combines call by value and call by result.

It performs two copies:

```text
procedure entry:
caller argument → callee parameter

procedure return:
callee parameter → caller argument
```

So the callee begins with the caller’s original value, may modify it locally, and then copies the final value back when the procedure returns.

Conceptual flow:

```text
Before call:
a = 3

Call entry:
x = copy of a

Inside callee:
x = 10

Return:
a = copy of x
```

This mechanism is implemented in Ada as `inout` parameters and is also valid for Fortran.

A key implementation issue is copying cost. Like call by value, it is efficient for small register-sized values but expensive for large objects.

---

## Call by Reference

**Call by reference** passes access to the caller’s actual variable, usually by passing its address.

Conceptually:

```text
caller variable
        ↑
callee formal parameter refers to same storage location
```

The callee can read and write the caller’s variable directly.

Example idea:

```cpp
void f(int& x) {
    x = 10;
}

int a = 3;
f(a);
// a becomes 10
```

Implementation:

```text
caller passes address of actual argument
callee uses that address to access or modify the variable
```

This is efficient for arrays because no array copy is needed. However, it can be inefficient for small scalar values because the value may need to live in memory rather than being passed purely in registers.

The textbook also mentions an important constant-passing problem. If a constant is passed by reference and the callee modifies it, the constant’s shared storage could be corrupted. A common solution is to copy the constant into a new anonymous temporary location and pass the address of that temporary.

Languages mentioned:

- Fortran
    
- C and C++ indirectly, because passing pointers by value can simulate reference-like behavior
    
- Algol 68 through address-like values
    

---

## Call by Name

**Call by name** is the most complex mechanism.

It is similar to call by reference because the callee can access the caller’s argument, but the key difference is:

```text
Call by reference:
address is computed once, at procedure entry

Call by name:
address is recomputed every time the parameter is used
```

This matters when the argument expression depends on a variable whose value changes.

Example idea:

```text
f(a[i], i)
```

If `i` changes inside `f`, then later uses of `a[i]` may refer to a different array element.

Implementation usually uses a **thunk**:

```text
parameter access
      ↓
call thunk
      ↓
compute current address of argument
      ↓
load/store through that address
```

A thunk is a parameterless helper procedure that computes the current address of the argument each time the parameter is accessed.

The textbook emphasizes that this mechanism can be expensive. However, simple cases such as passing a plain variable, a whole array, or a fixed array element can often be optimized into call by reference because the address does not change.

Call by name is mainly of historical importance and is associated with Algol 60.

---

## Label Parameters

Some languages allow **labels** to be passed as procedure arguments.

A label parameter can be used as the target of a `goto` inside the callee. To implement this, the compiler cannot pass only the code address. It must also pass enough stack-frame information to return to the correct activation context.

The textbook says label passing requires:

```text
1. code address of the label
2. dynamic link of the corresponding stack frame
```

When the callee executes a `goto` to a label parameter, the runtime system may need to perform one or more returns until it reaches the correct stack frame, then branch to the label’s code address.

Conceptually:

```text
goto label_parameter
        ↓
restore correct caller frame
        ↓
branch to label's code address
```

This feature is found in languages such as Algol 60 and Fortran.

---

## Comparison Table

|Discipline|Entry Action|Return Action|Can Callee Modify Caller Variable?|Main Cost|
|---|--:|--:|--:|---|
|Call by value|Copy value into callee|Nothing|No, except through pointers|Copying large objects|
|Call by result|Nothing|Copy result back|Yes, on return|Copy-back cost|
|Call by value-result|Copy in|Copy out|Yes, on return|Copy-in and copy-out|
|Call by reference|Pass address|Nothing special|Yes, immediately|Aliasing and memory access|
|Call by name|Recompute address on each use|Nothing special|Yes, depending on expression|Thunk calls and dynamic address computation|

---

## Compiler-Level Perspective

From a compiler writer’s perspective, parameter passing is a tradeoff between **semantic correctness** and **runtime efficiency**.

The major tradeoffs are:

```text
Small scalar values
    → best passed by value in registers

Large arrays or structures
    → often better passed by reference to avoid copying

Output-only values
    → call by result or explicit return values

Input-output values
    → call by value-result or call by reference

Expression-like parameters
    → call by name, but expensive
```

This is why real compilers often choose different low-level implementations depending on the type and size of the parameter.

For example, Fortran semantics allow either call by value-result or call by reference for each argument. This gives the compiler flexibility: small values can be handled efficiently with value-result, while arrays can be passed by reference to avoid copying.

---

## Key Takeaways

### 1. Parameter Passing Defines Caller-Callee Interaction

Each discipline defines how the caller’s data is exposed to the callee: by copying values, copying results, passing addresses, or recomputing argument locations.

### 2. Copying and Aliasing Are the Two Main Costs

Call by value and value-result may cause expensive copying. Call by reference avoids copying but introduces aliasing, which makes optimization harder.

### 3. Arrays Change the Cost Model

Passing a scalar by value is cheap. Passing a large array by value may be very expensive unless the compiler can optimize the copy away.

### 4. Call by Name Is Powerful but Expensive

Call by name recomputes the argument’s address on every use, usually through thunks. This gives unusual semantics but high runtime overhead.

### 5. Compiler Implementation Depends on Both Language Semantics and Target Architecture

The same source-level discipline may be implemented differently depending on register availability, stack layout, argument size, aliasing rules, and interprocedural analysis.