---
layout: single
title: "[Compiler] 5.7 Code Sharing and Position-Independent Code"
date: 2026-06-08 14:44:12 +0800
categories: Compiler
tags: [Compiler, Runtime-Support]
toc: true
toc_sticky: true
---

## Core Idea

Section 5.7 explains how compiled programs can share code at run time through **shared objects** and why this requires **position-independent code**, usually abbreviated as **PIC**.

So far, the textbook assumed a simple execution model:

```text
program code + statically linked libraries
        ↓
single executable image
        ↓
loaded into memory and executed
```

In that model, library routines are linked **before execution**. However, this causes space and maintenance problems because every executable may contain its own copy of library code.

Shared libraries solve this by allowing library code to be loaded and linked **dynamically** during execution, while the same code copy can be shared by multiple running programs. The textbook notes that shared libraries reduce both file-system duplication and memory duplication, and also allow a library implementation to be replaced without relinking every program that uses it, as long as the interface remains compatible.

---

## Why Code Sharing Is Useful

Code sharing mainly improves **space utilization** and **software maintenance**.

Instead of this:

```text
Program A → has its own copy of library code
Program B → has its own copy of library code
Program C → has its own copy of library code
```

we get this:

```text
Program A ┐
Program B ├──→ one shared library code copy
Program C ┘
```

The advantages are:

- only one copy of the shared library needs to exist in the file system,
    
- only one copy of the shared code needs to exist in memory,
    
- library bugs can be fixed by replacing the shared library, without relinking old executables.
    

This is especially valuable for large libraries, such as windowing systems, graphics libraries, or system libraries. Even though static linking may only include the routines actually used by a program, the textbook argues that shared libraries usually still win when many programs use the same large library.

---

## Static Linking vs Dynamic Linking

With **static linking**, the linker combines the program and library code before execution.

```text
Compile
  ↓
Static link
  ↓
Executable already contains needed library code
  ↓
Run
```

With **dynamic linking**, the executable records that it needs certain external routines or variables, but the actual library object may be loaded and connected later.

```text
Compile
  ↓
Pre-execution link check
  ↓
Run program
  ↓
Load/link shared object when needed
```

A subtle requirement is that dynamic linking should preserve the semantics of static linking as much as possible. In particular, the system should still be able to detect missing or multiply defined external symbols before execution when possible. The textbook describes this using a **table of contents** for each shared library: it lists entry points, external symbols, and the other shared libraries they depend on. This allows the pre-execution linker to check whether dynamic linking should succeed.

---

## Shared Objects, Not Just Shared Libraries

The textbook broadens the term from **shared library** to **shared object**.

A shared object is simply a unit of code/data that a programmer chooses to link at run time rather than before execution. It does not have to be a traditional library.

```text
shared library ⊂ shared object
```

So the more general model is:

```text
Executable
   + dynamically linked shared object A
   + dynamically linked shared object B
   + dynamically linked shared object C
```

This distinction matters because the mechanism is not limited to standard libraries. Any code unit can potentially be compiled, loaded, and linked as a shared object.

---

## Performance Cost of Shared Objects

Shared objects are not free. The textbook identifies two major costs:

1. **Run-time linking cost**  
    Some linking work is delayed until the program is running.
    
2. **Position-independent code overhead**  
    Shared objects must be compiled so that they can be loaded at different memory addresses in different processes.
    

However, on a multiprogrammed system, the cost may be offset by reduced working-set size. If many programs share the same code pages, the system may get better paging and cache behavior.

A useful way to think about the tradeoff:

```text
Shared objects
    ↓
slightly more expensive access/linking
    ↓
less duplicated code in memory
    ↓
potentially better paging/cache behavior
```

---

## Why Position-Independent Code Is Needed

A shared object cannot assume that it will always be loaded at one fixed address.

Different programs may have different memory layouts:

```text
Process 1:
[ main program ][ shared object X at address 0x4000 ]

Process 2:
[ larger main program ][ shared object X at address 0x9000 ]
```

Therefore, the same shared object must work correctly even when loaded at different addresses. This is the purpose of **position-independent code**.

The textbook states that position independence is needed so that each user of a shared object can map it to any address in memory. This creates four concrete problems:

1. how control is passed within an object,
    
2. how an object addresses its own external variables,
    
3. how control is passed between objects,
    
4. how an object addresses external variables belonging to other objects.
    

---

## Problem 1: Control Transfer Within One Shared Object

Control transfer **inside the same shared object** is usually easy.

Reason: although the shared object may be loaded at different absolute addresses, the **relative distance** between instructions inside the object is fixed.

Example:

```text
Shared object loaded at 0x4000:
function A → function B is +200 bytes away

Shared object loaded at 0x9000:
function A → function B is still +200 bytes away
```

So the compiler can use **PC-relative branches and calls**.

```text
current PC + relative offset = target address
```

This is naturally position-independent because the code does not need the absolute address of the target. The textbook notes that if the architecture does not provide PC-relative calls directly, the compiler can simulate them by constructing the target address from the current point and a relative offset.

---

## Problem 2: Accessing Variables Inside the Shared Object

Local variables are not the main issue.

They are usually stored:

- in registers, or
    
- in stack-frame locations accessed through registers.
    

So local variables are already private to each process and do not depend on fixed absolute addresses.

The real issue is **global or external variables**, because they are often accessed through absolute addresses in ordinary non-PIC code. In shared objects, absolute addresses are unsafe because the object may be loaded at different locations.

The common solution is the **Global Offset Table**, or **GOT**.

```text
Shared object code
    ↓
uses GOT pointer
    ↓
GOT entry contains actual address of global/external variable
    ↓
load/store through that address
```

The textbook explains that the GOT initially contains offsets for external symbols. When dynamic linking occurs, these offsets are turned into absolute addresses in the current process’s data space. Procedures then use a register, often called `gp`, to point to the GOT base.

Example idea:

```text
gp → base of GOT

GOT[a_offset] → actual address of global variable a
actual address of a → value of a
```

So accessing a global variable often becomes a two-step operation:

```text
load address of a from GOT
load value of a from that address
```

This is more expensive than direct addressing, but it allows the same code to work regardless of where the shared object is loaded.

---

## Problem 3: Control Transfer Between Shared Objects

Calling a routine in another shared object is harder than calling within the same object.

Inside one object, relative offsets are known. But between two shared objects, their relative positions are not known at compile time, and may not even be known when the main program first starts.

```text
Object A calls function f in Object B

But Object B may not be loaded yet.
Even if loaded, its address may differ per process.
```

The standard solution is a **stub**.

A stub is a small piece of code that acts as the call target first. The call goes to the stub, and the stub is responsible for reaching the real function.

```text
caller
  ↓
stub
  ↓
dynamic linker, if needed
  ↓
actual callee
```

The textbook describes organizing these stubs into a **Procedure Linkage Table**, or **PLT**. A PLT entry can initially invoke the dynamic linker. After the callee has been resolved, the PLT entry can be modified so that later calls go directly to the resolved routine.

Conceptually:

```text
First call:
call PLT[f]
    ↓
dynamic linker resolves f
    ↓
jump to real f

Later calls:
call PLT[f]
    ↓
jump directly to real f
```

---

## Problem 4: Accessing Variables in Other Shared Objects

Accessing another object’s data is similar to calling another object’s routine: the exact address is not known when compiling the shared object.

The solution again depends on indirection through tables such as the GOT.

```text
Object A needs variable x from Object B
    ↓
Object A uses GOT entry for x
    ↓
dynamic linker fills GOT entry with actual address
    ↓
Object A accesses x indirectly
```

This separates the compiled code from absolute addresses. The compiled instructions stay position-independent, while the dynamic linker patches or fills table entries at run time.

---

## GOT vs PLT

The two central mechanisms are:

```text
GOT = Global Offset Table
PLT = Procedure Linkage Table
```

They solve related but different problems.

|Mechanism|Main Purpose|Used For|
|---|---|---|
|GOT|stores resolved addresses of data symbols|global/external variable access|
|PLT|routes calls through stubs|external procedure calls|

Simple mental model:

```text
GOT answers: "Where is this variable?"
PLT answers: "How do I call this function?"
```

The GOT usually supports position-independent data access. The PLT usually supports dynamic function resolution and lazy binding.

---

## Procedure Variables and Shared Objects

There is one subtle issue with procedure addresses.

In languages like C, a program may take the address of a function, store it in a variable, and compare it with another function pointer.

This becomes tricky with shared objects:

```text
inside shared object:
address of f might look like address of f's real code

outside shared object:
address of f might look like address of f's PLT stub
```

If those are different, function-pointer comparison breaks.

The textbook’s solution is to use **procedure descriptors** uniformly. Instead of treating a procedure value as just a raw code address, the descriptor can contain the PLT entry address and the GOT address for the object containing the callee. This allows procedure variables and comparisons to work consistently across shared-object boundaries.

---

## Big Picture Flow

```text
Source program
    ↓
Compiler generates PIC
    ↓
Shared object contains:
    - PC-relative internal control transfers
    - GOT-based global/external data access
    - PLT-based external procedure calls
    ↓
Loader maps shared object at some address
    ↓
Dynamic linker resolves symbols
    ↓
Program runs using shared code
```

The key idea is that the code itself should not depend on fixed absolute addresses. Instead, it uses relative addressing and indirection tables.

---

## Key Takeaways

### 1. Shared objects reduce duplication

They allow multiple programs to share one copy of library/object code in memory and in the file system.

### 2. Dynamic linking delays binding

Some symbol resolution happens at run time rather than entirely before execution.

### 3. PIC makes sharing possible

Position-independent code allows the same compiled object to be loaded at different addresses in different processes.

### 4. GOT handles data addresses

The Global Offset Table provides a position-independent way to access global and external variables.

### 5. PLT handles procedure calls

The Procedure Linkage Table provides a position-independent and dynamically linkable way to call routines in other shared objects.

### 6. Indirection is the price of flexibility

PIC and dynamic linking add some overhead, but they enable memory sharing, easier library updates, and flexible run-time composition.