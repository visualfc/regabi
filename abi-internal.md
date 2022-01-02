# Go internal ABI specification

# Go 内部 ABI 规范

This document describes Go’s internal application binary interface
(ABI), known as ABIInternal.
Go's ABI defines the layout of data in memory and the conventions for
calling between Go functions.
This ABI is *unstable* and will change between Go versions.
If you’re writing assembly code, please instead refer to Go’s
[assembly documentation](/doc/asm.html), which describes Go’s stable
ABI, known as ABI0.

本文档描述 Go 的 内部应用程序接口 (ABI)，称为 ABIInternal.
Go 的 ABI 定义了数据在内存中的布局和 Go 函数调用规则。
这个ABI是 *不稳定的* 并且在 Go 版本之间会发生变化。
如果你正在写汇编代码，请参考 Go [汇编文档](/doc/asm.html)，它描述了Go的稳定性
ABI，也就是ABI0。

All functions defined in Go source follow ABIInternal.
However, ABIInternal and ABI0 functions are able to call each other
through transparent *ABI wrappers*, described in the [internal calling
convention proposal](https://golang.org/design/27539-internal-abi).

所有在 Go 源码中定义的函数都遵循 ABIInternal。
然而，ABIInternal 和 ABI0 函数可以通过透明的 *ABI wrappers* 互相调用，
相关描述在 [internal calling
convention proposal](https://golang.org/design/27539-internal-abi).

Go uses a common ABI design across all architectures.
We first describe the common ABI, and then cover per-architecture
specifics.

Go 在所有架构中使用通用 ABI 设计。我们首先描述通用 ABI，然后介绍每个体系结构细节。

*Rationale*: For the reasoning behind using a common ABI across
architectures instead of the platform ABI, see the [register-based Go
calling convention proposal](https://golang.org/design/40724-register-calling).

*基本原理*: 使用跨平台的通用 ABI 代替平台 ABI，参见 [register-based Go
calling convention proposal](https://golang.org/design/40724-register-calling).


## Memory layout

## 内存布局

Go's built-in types have the following sizes and alignments.
Many, though not all, of these sizes are guaranteed by the [language
specification](/doc/go_spec.html#Size_and_alignment_guarantees).
Those that aren't guaranteed may change in future versions of Go (for
example, we've considered changing the alignment of int64 on 32-bit).

Go 的内置类型具有如下的大小和对齐方式。
许多(尽管不是全部)大小是由[语言规范](/ doc / go_spec.html # Size_and_alignment_guarantees)。
保证的。在未来版本的围棋中，那些不能保证的规则可能会改变（例如，我们考虑过改变 int64 在32位上的对齐方式)。

| Type | 64-bit |       | 32-bit |       |
| ---  | ---    | ---   | ---    | ---   |
|      | Size   | Align | Size   | Align |
| bool, uint8, int8  | 1  | 1 | 1  | 1 |
| uint16, int16      | 2  | 2 | 2  | 2 |
| uint32, int32      | 4  | 4 | 4  | 4 |
| uint64, int64      | 8  | 8 | 8  | 4 |
| int, uint          | 8  | 8 | 4  | 4 |
| float32            | 4  | 4 | 4  | 4 |
| float64            | 8  | 8 | 8  | 4 |
| complex64          | 8  | 4 | 8  | 4 |
| complex128         | 16 | 8 | 16 | 4 |
| uintptr, *T, unsafe.Pointer | 8 | 8 | 4 | 4 |

The types `byte` and `rune` are aliases for `uint8` and `int32`,
respectively, and hence have the same size and alignment as these
types.

类型 `byte` 和 `rune` 分别是 `uint8` 和 `int32` 的别名，因此和对应类型具有相同的大小和对齐方式。

The layout of `map`, `chan`, and `func` types is equivalent to *T.

类型 `map`、 `chan`、 和 `func` 的布局等同于 *T.

To describe the layout of the remaining composite types, we first
define the layout of a *sequence* S of N fields with types
t<sub>1</sub>, t<sub>2</sub>, ..., t<sub>N</sub>.
We define the byte offset at which each field begins relative to a
base address of 0, as well as the size and alignment of the sequence
as follows:

为了描述其余复合类型的布局，我们首先定义一个包含 N 个字段的 *序列* S 的布局，字段类型为
t<sub>1</sub>, t<sub>2</sub>, ..., t<sub>N</sub>.
我们定义每个字段开始时相对于基址 0 的字节偏移量，以及序列的大小和对齐方式如下:

```
offset(S, i) = 0  if i = 1
             = align(offset(S, i-1) + sizeof(t_(i-1)), alignof(t_i))
alignof(S)   = 1  if N = 0
             = max(alignof(t_i) | 1 <= i <= N)
sizeof(S)    = 0  if N = 0
             = align(offset(S, N) + sizeof(t_N), alignof(S))
```

Where sizeof(T) and alignof(T) are the size and alignment of type T,
respectively, and align(x, y) rounds x up to a multiple of y.

其中 sizeof(T) 和 alignof(T) 分别表示类型 T 的大小和对齐方式，align(x, y) 将 x 舍入到 y 的倍数。

The `interface{}` type is a sequence of 1. a pointer to the runtime type
description for the interface's dynamic type and 2. an `unsafe.Pointer`
data field.

接口 `interface{}` 类型序列为 1. 一个指向描述接口动态类型的运行时类型的指针 2. 一个 `unsafe.Pointer` 数据字段。

Any other interface type (besides the empty interface) is a sequence
of 1. a pointer to the runtime "itab" that gives the method pointers and
the type of the data field and 2. an `unsafe.Pointer` data field.
An interface can be "direct" or "indirect" depending on the dynamic
type: a direct interface stores the value directly in the data field,
and an indirect interface stores a pointer to the value in the data
field.
An interface can only be direct if the value consists of a single
pointer word.

任何其他接口类型（除了空接口）都属于序列 1. 指向运行时 “itab” 的指针，该指针提供方法指针和数据字段的类型。 2. `unsafe.Pointer` 数据字段。
接口可以是“直接”或“间接”，具体取决于动态类型：直接接口将值直接存储在数据字段中，间接接口存储指向数据中值的指针。
只有当值由单个指针构成时，才是直接接口。

An array type `[N]T` is a sequence of N fields of type T.

数组类型 `[N]T` 是由 N 个类型为 T 的字段组成的序列。

The slice type `[]T` is a sequence of a `*[cap]T` pointer to the slice
backing store, an `int` giving the `len` of the slice, and an `int`
giving the `cap` of the slice.

切片类型 `[]T` 的序列包含一个 `*[cap]T` 的指针指向切片后台存储，一个 `int` 类型的 `len` 表示切片长度，
一个 `int` 类型 `cap` 表示切片容量。

The `string` type is a sequence of a `*[len]byte` pointer to the
string backing store, and an `int` giving the `len` of the string.

字符串类型 `string` 的序列包含一个 `*[len]byte` 指针指向字符串后台存储，和一个 `int` 类型的 `len` 表示字符串的长度。

A struct type `struct { f1 t1; ...; fM tM }` is laid out as the
sequence t1, ..., tM, tP, where tP is either:

- Type `byte` if sizeof(tM) = 0 and any of sizeof(t*i*) ≠ 0.
- Empty (size 0 and align 1) otherwise.

结构体类型 `struct{f1 t1；…；fM tM}` 作为序列 t1, ..., tM, tP, 其中 tP 是：

- 如果 sizeof(tM) == 0 并且任意 sizeof(t*i*) ≠ 0，则为 `byte` 类型。
- 否则为空 (大小为0，对齐为1)

The padding byte prevents creating a past-the-end pointer by taking
the address of the final, empty fN field.

填充字节以防止通过获取最后空的 fN 字段地址来一个创建越界指针。

Note that user-written assembly code should generally not depend on Go
type layout and should instead use the constants defined in
[`go_asm.h`](/doc/asm.html#data-offsets).

请注意，用户编写的汇编代码通常不应该依赖于 Go 类型布局，应该使用定义的常量 [`go_asm.h`](/doc/asm.html#data-offsets)。

## Function call argument and result passing

## 函数调用参数和结果传递

Function calls pass arguments and results using a combination of the
stack and machine registers.
Each argument or result is passed either entirely in registers or
entirely on the stack.
Because access to registers is generally faster than access to the
stack, arguments and results are preferentially passed in registers.
However, any argument or result that contains a non-trivial array or
does not fit entirely in the remaining available registers is passed
on the stack.


Function calls pass arguments and results using a combination of the
stack and machine registers.
Each argument or result is passed either entirely in registers or
entirely on the stack.
Because access to registers is generally faster than access to the
stack, arguments and results are preferentially passed in registers.
However, any argument or result that contains a non-trivial array or
does not fit entirely in the remaining available registers is passed
on the stack.

函数调用参数和结果使用堆栈和计算机寄存器的组合。
每个参数或结果要么完全在寄存器中传递，要么完全在堆栈上。
因为访问寄存器通常比访问堆栈要快，所以参数和结果优先在寄存器中传递。
但是，任何参数或结果包含非平凡数组或者不完全适合剩余可用寄存器的情况下使用堆栈传递。

Each architecture defines a sequence of integer registers and a
sequence of floating-point registers.
At a high level, arguments and results are recursively broken down
into values of base types and these base values are assigned to
registers from these sequences.

Arguments and results can share the same registers, but do not share
the same stack space.
Beyond the arguments and results passed on the stack, the caller also
reserves spill space on the stack for all register-based arguments
(but does not populate this space).

每个体系结构定义一系列整数寄存器和浮点寄存器序列。
在高层，参数和结果被递归地分解转换为基础类型值，并将这些基值指定给来自这些序列的寄存器。

The receiver, arguments, and results of function or method F are
assigned to registers or the stack using the following algorithm:

1. Let NI and NFP be the length of integer and floating-point register
   sequences defined by the architecture.
   Let I and FP be 0; these are the indexes of the next integer and
   floating-pointer register.
   Let S, the type sequence defining the stack frame, be empty.
1. If F is a method, assign F’s receiver.
1. For each argument A of F, assign A.
1. Add a pointer-alignment field to S. This has size 0 and the same
   alignment as `uintptr`.
1. Reset I and FP to 0.
1. For each result R of F, assign R.
1. Add a pointer-alignment field to S.
1. For each register-assigned receiver and argument of F, let T be its
   type and add T to the stack sequence S.
   This is the argument's (or receiver's) spill space and will be
   uninitialized at the call.
1. Add a pointer-alignment field to S.

函数或方法 F 的接收器、参数和结果是使用以下算法分配给寄存器或堆栈:

1. 设 NI 和 NFP 为由体系结构定义的整数和浮点寄存器序列的长度。
   设 I 和 FP 为 0 作为下一个整数和浮点指针寄存器的索引。
   设 S 为定义堆栈帧的类型序列，为空。
1. 如果 F 是方法，分配 F 的接收器。
1. 对于 F 的每个参数 A，分配 A。
1. 向 S 添加指针对齐字段，该字段的大小为0，对齐方式与 `uintptr` 相同。
1. 将 I 和 FP 重置为0。
1. 对于 F 的每个结果 R，分配 R。
1. 将指针对齐字段添加到 S。
1. 对于 F 每个寄存器分配的接收器和参数，设 T 为其类型，并将 T 添加到堆栈序列 S 中。
   这是参数(或接收器)的溢出空间，在调用时没有初始化。
1. 添加一个指针对齐字段到 S。

Assigning a receiver, argument, or result V of underlying type T works
as follows:

1. Remember I and FP.
1. If T has zero size, add T to the stack sequence S and return.
1. Try to register-assign V.
1. If step 2 failed, reset I and FP to the values from step 1, add T
   to the stack sequence S, and assign V to this field in S.

Register-assignment of a value V of underlying type T works as follows:

1. If T is a boolean or integral type that fits in an integer
   register, assign V to register I and increment I.
1. If T is an integral type that fits in two integer registers, assign
   the least significant and most significant halves of V to registers
   I and I+1, respectively, and increment I by 2
1. If T is a floating-point type and can be represented without loss
   of precision in a floating-point register, assign V to register FP
   and increment FP.
1. If T is a complex type, recursively register-assign its real and
   imaginary parts.
1. If T is a pointer type, map type, channel type, or function type,
   assign V to register I and increment I.
1. If T is a string type, interface type, or slice type, recursively
   register-assign V’s components (2 for strings and interfaces, 3 for
   slices).
1. If T is a struct type, recursively register-assign each field of V.
1. If T is an array type of length 0, do nothing.
1. If T is an array type of length 1, recursively register-assign its
   one element.
1. If T is an array type of length > 1, fail.
1. If I > NI or FP > NFP, fail.
1. If any recursive assignment above fails, fail.

The above algorithm produces an assignment of each receiver, argument,
and result to registers or to a field in the stack sequence.
The final stack sequence looks like: stack-assigned receiver,
stack-assigned arguments, pointer-alignment, stack-assigned results,
pointer-alignment, spill space for each register-assigned argument,
pointer-alignment.
The following diagram shows what this stack frame looks like on the
stack, using the typical convention where address 0 is at the bottom:

    +------------------------------+
    |             . . .            |
    | 2nd reg argument spill space |
    | 1st reg argument spill space |
    | <pointer-sized alignment>    |
    |             . . .            |
    | 2nd stack-assigned result    |
    | 1st stack-assigned result    |
    | <pointer-sized alignment>    |
    |             . . .            |
    | 2nd stack-assigned argument  |
    | 1st stack-assigned argument  |
    | stack-assigned receiver      |
    +------------------------------+ ↓ lower addresses

To perform a call, the caller reserves space starting at the lowest
address in its stack frame for the call stack frame, stores arguments
in the registers and argument stack fields determined by the above
algorithm, and performs the call.
At the time of a call, spill space, result stack fields, and result
registers are left uninitialized.
Upon return, the callee must have stored results to all result
registers and result stack fields determined by the above algorithm.

There are no callee-save registers, so a call may overwrite any
register that doesn’t have a fixed meaning, including argument
registers.

### Example

Consider the function `func f(a1 uint8, a2 [2]uintptr, a3 uint8) (r1
struct { x uintptr; y [2]uintptr }, r2 string)` on a 64-bit
architecture with hypothetical integer registers R0–R9.

On entry, `a1` is assigned to `R0`, `a3` is assigned to `R1` and the
stack frame is laid out in the following sequence:

    a2      [2]uintptr
    r1.x    uintptr
    r1.y    [2]uintptr
    a1Spill uint8
    a3Spill uint8
    _       [6]uint8  // alignment padding

In the stack frame, only the `a2` field is initialized on entry; the
rest of the frame is left uninitialized.

On exit, `r2.base` is assigned to `R0`, `r2.len` is assigned to `R1`,
and `r1.x` and `r1.y` are initialized in the stack frame.

There are several things to note in this example.
First, `a2` and `r1` are stack-assigned because they contain arrays.
The other arguments and results are register-assigned.
Result `r2` is decomposed into its components, which are individually
register-assigned.
On the stack, the stack-assigned arguments appear at lower addresses
than the stack-assigned results, which appear at lower addresses than
the argument spill area.
Only arguments, not results, are assigned a spill area on the stack.

### Rationale

Each base value is assigned to its own register to optimize
construction and access.
An alternative would be to pack multiple sub-word values into
registers, or to simply map an argument's in-memory layout to
registers (this is common in C ABIs), but this typically adds cost to
pack and unpack these values.
Modern architectures have more than enough registers to pass all
arguments and results this way for nearly all functions (see the
appendix), so there’s little downside to spreading base values across
registers.

Arguments that can’t be fully assigned to registers are passed
entirely on the stack in case the callee takes the address of that
argument.
If an argument could be split across the stack and registers and the
callee took its address, it would need to be reconstructed in memory,
a process that would be proportional to the size of the argument.

Non-trivial arrays are always passed on the stack because indexing
into an array typically requires a computed offset, which generally
isn’t possible with registers.
Arrays in general are rare in function signatures (only 0.7% of
functions in the Go 1.15 standard library and 0.2% in kubelet).
We considered allowing array fields to be passed on the stack while
the rest of an argument’s fields are passed in registers, but this
creates the same problems as other large structs if the callee takes
the address of an argument, and would benefit <0.1% of functions in
kubelet (and even these very little).

We make exceptions for 0 and 1-element arrays because these don’t
require computed offsets, and 1-element arrays are already decomposed
in the compiler’s SSA representation.

The ABI assignment algorithm above is equivalent to Go’s stack-based
ABI0 calling convention if there are zero architecture registers.
This is intended to ease the transition to the register-based internal
ABI and make it easy for the compiler to generate either calling
convention.
An architecture may still define register meanings that aren’t
compatible with ABI0, but these differences should be easy to account
for in the compiler.

The assignment algorithm assigns zero-sized values to the stack
(assignment step 2) in order to support ABI0-equivalence.
While these values take no space themselves, they do result in
alignment padding on the stack in ABI0.
Without this step, the internal ABI would register-assign zero-sized
values even on architectures that provide no argument registers
because they don't consume any registers, and hence not add alignment
padding to the stack.

The algorithm reserves spill space for arguments in the caller’s frame
so that the compiler can generate a stack growth path that spills into
this reserved space.
If the callee has to grow the stack, it may not be able to reserve
enough additional stack space in its own frame to spill these, which
is why it’s important that the caller do so.
These slots also act as the home location if these arguments need to
be spilled for any other reason, which simplifies traceback printing.

There are several options for how to lay out the argument spill space.
We chose to lay out each argument according to its type's usual memory
layout but to separate the spill space from the regular argument
space.
Using the usual memory layout simplifies the compiler because it
already understands this layout.
Also, if a function takes the address of a register-assigned argument,
the compiler must spill that argument to memory in its usual memory
layout and it's more convenient to use the argument spill space for
this purpose.

Alternatively, the spill space could be structured around argument
registers.
In this approach, the stack growth spill path would spill each
argument register to a register-sized stack word.
However, if the function takes the address of a register-assigned
argument, the compiler would have to reconstruct it in memory layout
elsewhere on the stack.

The spill space could also be interleaved with the stack-assigned
arguments so the arguments appear in order whether they are register-
or stack-assigned.
This would be close to ABI0, except that register-assigned arguments
would be uninitialized on the stack and there's no need to reserve
stack space for register-assigned results.
We expect separating the spill space to perform better because of
memory locality.
Separating the space is also potentially simpler for `reflect` calls
because this allows `reflect` to summarize the spill space as a single
number.
Finally, the long-term intent is to remove reserved spill slots
entirely – allowing most functions to be called without any stack
setup and easing the introduction of callee-save registers – and
separating the spill space makes that transition easier.

## Closures

A func value (e.g., `var x func()`) is a pointer to a closure object.
A closure object begins with a pointer-sized program counter
representing the entry point of the function, followed by zero or more
bytes containing the closed-over environment.

Closure calls follow the same conventions as static function and
method calls, with one addition. Each architecture specifies a
*closure context pointer* register and calls to closures store the
address of the closure object in the closure context pointer register
prior to the call.

## Software floating-point mode

In "softfloat" mode, the ABI simply treats the hardware as having zero
floating-point registers.
As a result, any arguments containing floating-point values will be
passed on the stack.

*Rationale*: Softfloat mode is about compatibility over performance
and is not commonly used.
Hence, we keep the ABI as simple as possible in this case, rather than
adding additional rules for passing floating-point values in integer
registers.

## Architecture specifics

This section describes per-architecture register mappings, as well as
other per-architecture special cases.

### amd64 architecture

The amd64 architecture uses the following sequence of 9 registers for
integer arguments and results:

    RAX, RBX, RCX, RDI, RSI, R8, R9, R10, R11

It uses X0 – X14 for floating-point arguments and results.

*Rationale*: These sequences are chosen from the available registers
to be relatively easy to remember.

Registers R12 and R13 are permanent scratch registers.
R15 is a scratch register except in dynamically linked binaries.

*Rationale*: Some operations such as stack growth and reflection calls
need dedicated scratch registers in order to manipulate call frames
without corrupting arguments or results.

Special-purpose registers are as follows:

| Register | Call meaning | Return meaning | Body meaning |
| --- | --- | --- | --- |
| RSP | Stack pointer | Same | Same |
| RBP | Frame pointer | Same | Same |
| RDX | Closure context pointer | Scratch | Scratch |
| R12 | Scratch | Scratch | Scratch |
| R13 | Scratch | Scratch | Scratch |
| R14 | Current goroutine | Same | Same |
| R15 | GOT reference temporary if dynlink | Same | Same |
| X15 | Zero value | Same | Scratch |

*Rationale*: These register meanings are compatible with Go’s
stack-based calling convention except for R14 and X15, which will have
to be restored on transitions from ABI0 code to ABIInternal code.
In ABI0, these are undefined, so transitions from ABIInternal to ABI0
can ignore these registers.

*Rationale*: For the current goroutine pointer, we chose a register
that requires an additional REX byte.
While this adds one byte to every function prologue, it is hardly ever
accessed outside the function prologue and we expect making more
single-byte registers available to be a net win.

*Rationale*: We could allow R14 (the current goroutine pointer) to be
a scratch register in function bodies because it can always be
restored from TLS on amd64.
However, we designate it as a fixed register for simplicity and for
consistency with other architectures that may not have a copy of the
current goroutine pointer in TLS.

*Rationale*: We designate X15 as a fixed zero register because
functions often have to bulk zero their stack frames, and this is more
efficient with a designated zero register.

*Implementation note*: Registers with fixed meaning at calls but not
in function bodies must be initialized by "injected" calls such as
signal-based panics.

#### Stack layout

The stack pointer, RSP, grows down and is always aligned to 8 bytes.

The amd64 architecture does not use a link register.

A function's stack frame is laid out as follows:

    +------------------------------+
    | return PC                    |
    | RBP on entry                 |
    | ... locals ...               |
    | ... outgoing arguments ...   |
    +------------------------------+ ↓ lower addresses

The "return PC" is pushed as part of the standard amd64 `CALL`
operation.
On entry, a function subtracts from RSP to open its stack frame and
saves the value of RBP directly below the return PC.
A leaf function that does not require any stack space may omit the
saved RBP.

The Go ABI's use of RBP as a frame pointer register is compatible with
amd64 platform conventions so that Go can inter-operate with platform
debuggers and profilers.

#### Flags

The direction flag (D) is always cleared (set to the “forward”
direction) at a call.
The arithmetic status flags are treated like scratch registers and not
preserved across calls.
All other bits in RFLAGS are system flags.

At function calls and returns, the CPU is in x87 mode (not MMX
technology mode).

*Rationale*: Go on amd64 does not use either the x87 registers or MMX
registers. Hence, we follow the SysV platform conventions in order to
simplify transitions to and from the C ABI.

At calls, the MXCSR control bits are always set as follows:

| Flag | Bit | Value | Meaning |
| --- | --- | --- | --- |
| FZ | 15 | 0 | Do not flush to zero |
| RC | 14/13 | 0 (RN) | Round to nearest |
| PM | 12 | 1 | Precision masked |
| UM | 11 | 1 | Underflow masked |
| OM | 10 | 1 | Overflow masked |
| ZM | 9 | 1 | Divide-by-zero masked |
| DM | 8 | 1 | Denormal operations masked |
| IM | 7 | 1 | Invalid operations masked |
| DAZ | 6 | 0 | Do not zero de-normals |

The MXCSR status bits are callee-save.

*Rationale*: Having a fixed MXCSR control configuration allows Go
functions to use SSE operations without modifying or saving the MXCSR.
Functions are allowed to modify it between calls (as long as they
restore it), but as of this writing Go code never does.
The above fixed configuration matches the process initialization
control bits specified by the ELF AMD64 ABI.

The x87 floating-point control word is not used by Go on amd64.

## Future directions

### Spill path improvements

The ABI currently reserves spill space for argument registers so the
compiler can statically generate an argument spill path before calling
into `runtime.morestack` to grow the stack.
This ensures there will be sufficient spill space even when the stack
is nearly exhausted and keeps stack growth and stack scanning
essentially unchanged from ABI0.

However, this wastes stack space (the median wastage is 16 bytes per
call), resulting in larger stacks and increased cache footprint.
A better approach would be to reserve stack space only when spilling.
One way to ensure enough space is available to spill would be for
every function to ensure there is enough space for the function's own
frame *as well as* the spill space of all functions it calls.
For most functions, this would change the threshold for the prologue
stack growth check.
For `nosplit` functions, this would change the threshold used in the
linker's static stack size check.

Allocating spill space in the callee rather than the caller may also
allow for faster reflection calls in the common case where a function
takes only register arguments, since it would allow reflection to make
these calls directly without allocating any frame.

The statically-generated spill path also increases code size.
It is possible to instead have a generic spill path in the runtime, as
part of `morestack`.
However, this complicates reserving the spill space, since spilling
all possible register arguments would, in most cases, take
significantly more space than spilling only those used by a particular
function.
Some options are to spill to a temporary space and copy back only the
registers used by the function, or to grow the stack if necessary
before spilling to it (using a temporary space if necessary), or to
use a heap-allocated space if insufficient stack space is available.
These options all add enough complexity that we will have to make this
decision based on the actual code size growth caused by the static
spill paths.

### Clobber sets

As defined, the ABI does not use callee-save registers.
This significantly simplifies the garbage collector and the compiler's
register allocator, but at some performance cost.
A potentially better balance for Go code would be to use *clobber
sets*: for each function, the compiler records the set of registers it
clobbers (including those clobbered by functions it calls) and any
register not clobbered by function F can remain live across calls to
F.

This is generally a good fit for Go because Go's package DAG allows
function metadata like the clobber set to flow up the call graph, even
across package boundaries.
Clobber sets would require relatively little change to the garbage
collector, unlike general callee-save registers.
One disadvantage of clobber sets over callee-save registers is that
they don't help with indirect function calls or interface method
calls, since static information isn't available in these cases.

### Large aggregates

Go encourages passing composite values by value, and this simplifies
reasoning about mutation and races.
However, this comes at a performance cost for large composite values.
It may be possible to instead transparently pass large composite
values by reference and delay copying until it is actually necessary.

## Appendix: Register usage analysis

In order to understand the impacts of the above design on register
usage, we
[analyzed](https://github.com/aclements/go-misc/tree/master/abi) the
impact of the above ABI on a large code base: cmd/kubelet from
[Kubernetes](https://github.com/kubernetes/kubernetes) at tag v1.18.8.

The following table shows the impact of different numbers of available
integer and floating-point registers on argument assignment:

```
|      |        |       |      stack args |          spills |     stack total |
| ints | floats | % fit | p50 | p95 | p99 | p50 | p95 | p99 | p50 | p95 | p99 |
|    0 |      0 |  6.3% |  32 | 152 | 256 |   0 |   0 |   0 |  32 | 152 | 256 |
|    0 |      8 |  6.4% |  32 | 152 | 256 |   0 |   0 |   0 |  32 | 152 | 256 |
|    1 |      8 | 21.3% |  24 | 144 | 248 |   8 |   8 |   8 |  32 | 152 | 256 |
|    2 |      8 | 38.9% |  16 | 128 | 224 |   8 |  16 |  16 |  24 | 136 | 240 |
|    3 |      8 | 57.0% |   0 | 120 | 224 |  16 |  24 |  24 |  24 | 136 | 240 |
|    4 |      8 | 73.0% |   0 | 120 | 216 |  16 |  32 |  32 |  24 | 136 | 232 |
|    5 |      8 | 83.3% |   0 | 112 | 216 |  16 |  40 |  40 |  24 | 136 | 232 |
|    6 |      8 | 87.5% |   0 | 112 | 208 |  16 |  48 |  48 |  24 | 136 | 232 |
|    7 |      8 | 89.8% |   0 | 112 | 208 |  16 |  48 |  56 |  24 | 136 | 232 |
|    8 |      8 | 91.3% |   0 | 112 | 200 |  16 |  56 |  64 |  24 | 136 | 232 |
|    9 |      8 | 92.1% |   0 | 112 | 192 |  16 |  56 |  72 |  24 | 136 | 232 |
|   10 |      8 | 92.6% |   0 | 104 | 192 |  16 |  56 |  72 |  24 | 136 | 232 |
|   11 |      8 | 93.1% |   0 | 104 | 184 |  16 |  56 |  80 |  24 | 128 | 232 |
|   12 |      8 | 93.4% |   0 | 104 | 176 |  16 |  56 |  88 |  24 | 128 | 232 |
|   13 |      8 | 94.0% |   0 |  88 | 176 |  16 |  56 |  96 |  24 | 128 | 232 |
|   14 |      8 | 94.4% |   0 |  80 | 152 |  16 |  64 | 104 |  24 | 128 | 232 |
|   15 |      8 | 94.6% |   0 |  80 | 152 |  16 |  64 | 112 |  24 | 128 | 232 |
|   16 |      8 | 94.9% |   0 |  16 | 152 |  16 |  64 | 112 |  24 | 128 | 232 |
|    ∞ |      8 | 99.8% |   0 |   0 |   0 |  24 | 112 | 216 |  24 | 120 | 216 |
```

The first two columns show the number of available integer and
floating-point registers.
The first row shows the results for 0 integer and 0 floating-point
registers, which is equivalent to ABI0.
We found that any reasonable number of floating-point registers has
the same effect, so we fixed it at 8 for all other rows.

The “% fit” column gives the fraction of functions where all arguments
and results are register-assigned and no arguments are passed on the
stack.
The three “stack args” columns give the median, 95th and 99th
percentile number of bytes of stack arguments.
The “spills” columns likewise summarize the number of bytes in
on-stack spill space.
And “stack total” summarizes the sum of stack arguments and on-stack
spill slots.
Note that these are three different distributions; for example,
there’s no single function that takes 0 stack argument bytes, 16 spill
bytes, and 24 total stack bytes.

From this, we can see that the fraction of functions that fit entirely
in registers grows very slowly once it reaches about 90%, though
curiously there is a small minority of functions that could benefit
from a huge number of registers.
Making 9 integer registers available on amd64 puts it in this realm.
We also see that the stack space required for most functions is fairly
small.
While the increasing space required for spills largely balances out
the decreasing space required for stack arguments as the number of
available registers increases, there is a general reduction in the
total stack space required with more available registers.
This does, however, suggest that eliminating spill slots in the future
would noticeably reduce stack requirements.
