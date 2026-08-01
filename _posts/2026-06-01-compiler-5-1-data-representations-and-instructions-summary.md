---
layout: single
title: "[Compiler] 5.1 Data Representations and Instructions Summary"
date: 2026-06-01 15:08:16 +0800
categories: Compiler
tags: [Compiler, Runtime-Support]
toc: true
toc_sticky: true
---

> **Source note:** This article is a study note based on Section 5.1 of Steven S. Muchnick, [*Advanced Compiler Design and Implementation*](https://shop.elsevier.com/books/advanced-compiler-design-and-implementation/muchnick/978-0-08-049871-3), Morgan Kaufmann, 1997 (ISBN 978-1-55860-320-2). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Core Idea

Section 5.1 explains how a compiler maps **source-language data types** into **run-time machine representations**.

A high-level language may provide integers, characters, arrays, records, strings, sets, pointers, and other structured data types. However, the target machine usually only provides a smaller set of primitive operations: loads, stores, arithmetic, comparisons, branches, and sometimes specialized string or bit-field instructions.

```text
Source-level type
        ↓
Compiler representation decision
        ↓
Memory layout + machine instructions
        ↓
Executable run-time behavior
```

The central compiler question is:

```text
How should this source-language value be stored,
and which target-machine instructions should manipulate it?
```

This section belongs to Chapter 5, **Run-Time Support**, whose table of contents places “Data Representations and Instructions” before register usage, stack frames, parameter passing, procedure linkage, and position-independent code.

---

## Scalar Types

Scalar types are values that usually fit into one machine word or a small fixed number of words.

Typical scalar types include:

```text
integer
character
floating-point value
enumerated value
Boolean
pointer
```

The compiler prefers to map these directly to hardware-supported formats because that gives the simplest instruction sequence.

```text
int x;
x = x + 1;
```

can often become:

```text
load x
add 1
store x
```

The important point is that “simple source type” does not always mean “single instruction.” If the source type is smaller, larger, or more semantically complex than the hardware type, the compiler must synthesize the operation.

```text
Source scalar type
        ↓
Native machine type?
        ↓
Yes → direct instruction
No  → masking / extension / multiword code / runtime helper
```

---

## Integers

Integers are usually represented using the target machine’s native integer format. Most architectures support ordinary word-sized integer loads, stores, arithmetic, and comparisons.

For smaller integer types, such as bytes and halfwords, the compiler may need sign-extension or zero-extension.

```text
8-bit signed integer in memory
        ↓ load byte
1111 1010
        ↓ sign-extend
1111 1111 1111 1111 1111 1111 1111 1010
```

For larger integers, a single source-level operation may require several machine instructions.

```text
64-bit integer on a 32-bit machine

High word        Low word
+---------+     +---------+
| upper32 |     | lower32 |
+---------+     +---------+

Addition:
lower32 add
carry propagation
upper32 add-with-carry
```

So integer representation affects both correctness and code generation complexity.

---

## Characters

Characters are usually represented as small integer-like values. Older systems often used one byte per character, but wider character encodings are needed for languages with large character sets, such as Chinese and Japanese writing systems.

Basic character operations are usually simple:

```text
load character
store character
compare character
```

Conceptually:

```text
char c = 'A';

memory byte
+----------+
| 0x41     |
+----------+
```

If the language uses wider characters:

```text
wide character

+----------+----------+
| byte 0   | byte 1   |
+----------+----------+
```

The compiler must choose instructions appropriate to the character width: byte load/store, halfword load/store, or a larger representation.

---

## Floating-Point Values

Floating-point values are usually represented using IEEE-style formats. Typical forms include:

```text
single precision
double precision
extended precision
```

A simplified floating-point representation looks like:

```text
+------+----------+-----------------------+
| sign | exponent | fraction / mantissa   |
+------+----------+-----------------------+
```

Floating-point compilation is more complicated than integer compilation because the compiler must preserve not only arithmetic results, but also floating-point semantics.

```text
Floating-point operation
        ↓
hardware instruction?
        ↓
rounding behavior
        ↓
exception behavior
        ↓
possible software assistance
```

The text notes that architectures differ in their support for single, double, and extended precision; for many architectures, IEEE-mandated exceptional values and exceptions require some software assistance.

---

## Enumerations and Booleans

Enumerations are usually represented as consecutive unsigned integers. For example:

```text
enum Color {
    Red,
    Orange,
    Yellow,
    Green
}
```

can be represented as:

```text
Red     → 0
Orange  → 1
Yellow  → 2
Green   → 3
```

The required operations are mostly:

```text
load
store
compare
```

Booleans are language-dependent:

```text
Pascal / Modula-2 / Ada → Boolean as an enumerated type
C                       → Boolean-like value as integer
Fortran 77              → separate logical type
```

Visualization:

```text
Boolean representation examples

true / false as enum:
false → 0
true  → 1

C-style integer truth:
0     → false
nonzero→ true
```

The text explicitly notes that enumerated values are generally consecutive unsigned integers, while Booleans may be represented as enumerations, integers, or separate types depending on the language.

---

## Arrays

Arrays are represented as contiguous storage blocks. A multidimensional array must be converted into a one-dimensional memory layout.

The two major layouts are:

```text
Row-major order     → common in C-like and Pascal-like languages
Column-major order  → used by Fortran
```

### Row-Major Layout

For a Pascal-style array:

```pascal
var a: array [1..10, 0..5] of integer
```

there are:

```text
10 × 6 = 60 integer elements
```

In row-major order:

```text
a[1,0]  a[1,1]  a[1,2]  a[1,3]  a[1,4]  a[1,5]
a[2,0]  a[2,1]  a[2,2]  a[2,3]  a[2,4]  a[2,5]
...
a[10,0] a[10,1] a[10,2] a[10,3] a[10,4] a[10,5]
```

Memory layout:

```text
word index:   0       1       2       3       4       5       6
element:    a[1,0] a[1,1] a[1,2] a[1,3] a[1,4] a[1,5] a[2,0]
```

The text gives this exact idea: `a[1,0]` is in the zeroth word, `a[1,1]` in the first word, `a[2,0]` in the sixth word, and `a[10,5]` in the 59th word.

### Address Calculation

For array access, the compiler generates address arithmetic.

```text
array element address
        =
base address
        +
linearized index × element size
```

Example intuition:

```text
a[i, j]

row-major:
offset = ((i - lower_i) × row_size + (j - lower_j)) × element_size
```

Visualization:

```text
Source access:
a[i, j]

Compiler lowering:
base(a)
  + row_offset(i)
  + column_offset(j)

Machine-level:
load/store [base + computed_offset]
```

Some architectures provide scaled-index addressing or base-register updating, which can reduce the number of instructions needed for repeated array accesses. The text mentions architectures such as POWER and PA-RISC in this context.

---

## Records and Structures

Records and structures group multiple named fields into one object. Machines usually do not directly understand “record” as a primitive operation, so the compiler must decide the field layout.

There are two major layout strategies:

```text
unpacked record
packed record
```

### Unpacked Record

An unpacked record respects alignment boundaries.

```c
struct s1 {
    int large1;
    short int small1;
};
```

Possible layout:

```text
+--------------------+----------+----------+
| large1: 32 bits    | small1   | padding  |
|                    | 16 bits  | 16 bits  |
+--------------------+----------+----------+
```

This usually makes access efficient because fields line up with natural machine loads and stores.

### Packed Record

A packed record stores fields adjacent to each other, even if that creates awkward access boundaries.

```c
struct s2 {
    int large2: 18;
    int small2: 10;
};
```

Possible layout:

```text
+------------------+------------+------+
| large2: 18 bits  | small2: 10 | pad  |
+------------------+------------+------+
```

Packed layout saves space, but field access may require extra operations:

```text
load word
        ↓
shift
        ↓
mask
        ↓
extract field
```

For writing a packed field:

```text
load old word
        ↓
clear target bits
        ↓
insert new field bits
        ↓
store updated word
```

The text’s Figures 5.1 and 5.2 compare an unpacked C struct and a packed bit-field struct, showing that the unpacked object may require a doubleword while the packed object may fit into a word.

---

## Pointers

Pointers usually occupy one word or one doubleword.

A pointer contains an address:

```text
pointer variable p

+----------------------+
| address of object x  |
+----------------------+
```

Dereferencing a pointer requires two conceptual steps:

```text
p
↓
load address stored in p
↓
use that address to access the object
```

Machine-level intuition:

```text
load r1, [p]      ; r1 = address stored in p
load r2, [r1]     ; r2 = object pointed to by p
```

For C and C++, pointer arithmetic is also part of the representation problem.

```c
p + 3
```

does not usually mean “add 3 bytes.” It means:

```text
p + 3 × sizeof(*p)
```

Visualization:

```text
int* p

p       p+1     p+2     p+3
↓       ↓       ↓       ↓
+----+  +----+  +----+  +----+
|int |  |int |  |int |  |int |
+----+  +----+  +----+  +----+
  4B     4B      4B      4B
```

The text notes that pointers are generally word- or doubleword-sized and are supported by loads, stores, and comparisons; the referenced object is accessed by loading the pointer into a register and using that register to form a memory address.

---

## Character Strings

Character strings are represented differently depending on the source language.

Two common forms are:

```text
length-counted string
null-terminated string
```

### Length-Counted String

Used by languages such as Pascal-style systems:

```text
+--------+------+------+------+------+
| length | c0   | c1   | c2   | ...  |
+--------+------+------+------+------+
```

This makes length access cheap:

```text
length(s) → read length field
```

### Null-Terminated String

Used by C:

```text
+------+------+------+------+------+
| c0   | c1   | c2   | ...  | '\0' |
+------+------+------+------+------+
```

This makes the string compact and simple, but length requires scanning:

```text
strlen(s)

start
  ↓
read char
  ↓
is char == '\0'?
  ↓ no
advance pointer
  ↓
repeat
```

Visualization:

```text
C string "cat"

+-----+-----+-----+------+
| 'c' | 'a' | 't' | '\0' |
+-----+-----+-----+------+
```

Some architectures provide string-specific instructions for move, compare, or scan operations. Others implement these operations using ordinary loops over loads and stores.

---

## Sets

Sets are often represented using bit strings. Each possible element corresponds to one bit.

```text
bit = 1 → element is present
bit = 0 → element is absent
```

Example:

```text
Set universe:
{red, orange, yellow, green, blue}

Set value:
{red, yellow, blue}

Bit vector:
red orange yellow green blue
 1     0      1      0    1
```

Memory representation:

```text
+---+---+---+---+---+
| 1 | 0 | 1 | 0 | 1 |
+---+---+---+---+---+
```

Set operations become bit operations:

```text
union        → bitwise OR
intersection → bitwise AND
difference   → bitwise AND-NOT
membership   → test one bit
```

Visualization:

```text
A = {red, yellow}
    1 0 1 0 0

B = {yellow, blue}
    0 0 1 0 1

A ∪ B
    1 0 1 0 1
```

For sparse sets, bit vectors can waste space. A sparse representation stores only present elements:

```text
Dense bit vector:
[1,0,0,0,0,0,0,1,0,0,0,1]

Sparse list:
[3 elements: 0, 7, 11]
```

This creates a tradeoff:

```text
bit vector
    → fast operations, more space for sparse sets

sparse list
    → compact for sparse sets, slower membership/union operations
```

---

## Composite and Rich Types

Many higher-level types can be represented by combining simpler layouts.

Examples:

```text
complex number
        ↓
record { real_part, imaginary_part }

rational number
        ↓
record { numerator, denominator }

array of records
        ↓
contiguous sequence of record layouts
```

Visualization:

```text
complex z = 3.0 + 4.0i

+----------------+----------------+
| real: 3.0      | imag: 4.0      |
+----------------+----------------+
```

For an array of records:

```text
struct Point {
    int x;
    int y;
};

Point points[3];
```

Memory can look like:

```text
+------+------+------+------+------+------+
| x0   | y0   | x1   | y1   | x2   | y2   |
+------+------+------+------+------+------+
```

This shows why data representation affects optimization: array traversal, field access, cache behavior, and instruction selection all depend on the chosen layout.

---

## Compiler Engineering View

The compiler’s job is not just to “store values.” It must choose representations that make source-language semantics executable on a real machine.

```text
Language semantics
        ↓
Type representation
        ↓
Memory layout
        ↓
Address calculation
        ↓
Instruction selection
        ↓
Runtime performance
```

A compact representation may save memory but require more instructions. A naturally aligned representation may use more space but allow direct loads and stores.

```text
Space-efficient layout
        ↔
Instruction-efficient layout
```

This is the core engineering tradeoff of Section 5.1.

---

## Key Takeaways

Data representation is the bridge between source-language types and target-machine instructions.

Scalar types often map directly to hardware-supported values, but differences in size, signedness, floating-point semantics, or language rules may require extra code.

Arrays require linearization, so the compiler must convert multidimensional subscripts into byte offsets.

Records require layout decisions, especially around alignment and packing.

Pointers are usually machine addresses, but source-language pointer arithmetic requires scaling by element size.

Strings and sets are language-dependent abstractions whose runtime representation strongly affects operation cost.

The central idea can be summarized as:

```text
A compiler does not merely translate operations.
It also translates data models.
```

Without correct data representation, even perfectly selected instructions would compute the wrong program behavior.