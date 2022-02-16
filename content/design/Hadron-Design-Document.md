---
title: Hadron Design Document
linkTitle: Hadron Design Document
author: Luke Nihlen
type: "posts"
---

{{< toc >}}

# Work In Progress

The Hadron code repository has several old and out-of-date documents with notes on the design, so I'm systematically
deleting those notes as I add updated documentation here. Consider this the authoritative source of Hadron design
documentation. Like Hadron, it's a work in progress. PRs welcome.

# Runtime Environment

During compilation, Hadron uses the [Lightening](https://gitlab.com/wingo/lightening) library to emit machine
instructions into an executable buffer. Lightening is a fork of GNU Lightning, which provides an interface with a
[generic set](https://www.gnu.org/software/lightning/manual/lightning.html#The-instruction-set) of CPU
instructions that it can translate into appropriate instructions specific to a given CPU instruction set.

Lightening allows Hadron to support different CPU architectures without changing the CPU instruction emission part of
the compilation pipeline. As part of this abstraction, Lightening renames the CPU registers generically as `GPR0`,
`GPR1`, and so on to the architecture-specific maximum. The GPR acronym is short for General Purpose Register, and these
registers may hold integers, pointers, or any other data type. The GPRs are the same size as the CPU architecture,
meaning on 64 bit CPUs, each GPR is 64 bits.

For floating-point operations, Lightening also provides `FP0`, `FP1`, and so on to the maximum number of floating point
registers on the CPU. Lightening has generic instructions to move data to and from a GPR to an FP and supports
mathematical operations on the Floating Point registers.

The [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) standards contain details specific to operating
systems and CPU architectures. These add complexity and overhead to Hadron-compiled code, which only needs to pass
messages between compiled classes. As a result, Hadron does not follow the ABI and instead maintains a separate stack
only for Hadron code. Hadron reserves 3 GPRs, `GPR0`, `GPR1`, and `GPR2`, as pointers to the `ThreadContext` structure
and the Hadron frame and stack pointers. All remaining registers are *caller-save*, meaning the callee can
change any of their values at will.

Hadron can transition from compiled C++ code into JIT bytecode and back out again via a *trampoline*,
code responsible for maintaining the C++ program stack and all registers, and then jumping to the Hadron
bytecode. The transitions between stacks are relatively expensive, so we take care to minimize them.

## The Stack Frame

Hadron allocates large-size `Frame` objects and adds them to the root set for scanning during garbage collection. The
`Frame` objects are not contiguous but rather individual chunks of memory with room for many stack frames in each. Some
compiler literature calls these "Stacklets." Excepting register spills, Hadron saves all values on the stack in `Slot`
format, meaning they are all 8 bytes with the header type tags included.

Within a Stacklet, stack addresses grow upward, starting with the value in the frame pointer and ending with the
register spill area past the stack pointer. Callee code expects a stack layout as follows:

| frame pointer | contents                           | stack pointer |
|---------------|------------------------------------|---------------|
| `fp`          | Caller Frame Pointer               |               |
| `fp` + 1      | Caller Stack Pointer               |               |
| `fp` + 2      | Caller Return Address              |               |
| `fp` + 3      | Message Selector Symbol            |               |
|    ...        | < dispatch work area >             |  ...          |
|               | Argument n - 1                     | `sp` - n      |
|               |  ...                               |  ...          |
|               | Argument 1                         | `sp` - 2      |
|               | Argument 0 (`this`) / Return Value | `sp` - 1      |
|               | < register spill area>             | `sp`          |

The dispatch work area is variable size, which we account for by maintaining two separate pointers into the stack. The
frame pointer indicates the beginning of the stack frame, and the stack pointer indicates the division between the
in-order argument values and the register spill area, a temporary area of memory used to save register values during
method calls and register allocation overflow.

Method calls always include a `this` pointer provided as the first argument. As `this` is also the default return value
for messages, we re-use this slot to store the return value of the message, avoiding an additional copy in those cases.

## Preparing the Stack for Message Dispatch

The caller code prepares a new stack frame and copies the arguments, both in-order and keyword-based, into the new stack
frame. As SuperCollider is a dynamic programming language, we don't know the message's intended recipient at compile
time. That means we don't know how many arguments the callee code expects or the names of those arguments.

The new stack frame starts just past the register spill area for the caller stack frame, and the caller saves its frame
pointer, stack pointer, and return address here. The caller then builds the dispatch work area by copying the message
selector symbol, the keyword count, and in-order argument count onto the stack, followed by any keyword/value pairs like
so:

| frame pointer | contents                           | stack pointer |
|---------------|------------------------------------|---------------|
| `fp`          | Caller Frame Pointer               |               |
| `fp` + 1      | Caller Stack Pointer               |               |
| `fp` + 2      | Caller Return Address              |               |
| `fp` + 3      | Message Selector Symbol            |               |
| `fp` + 4      | Number of In-order Arguments       |               |
| `fp` + 5      | Number of Keyword Arguments        |               |
| `fp` + 6      | Keyword Argument 0 Keyword         |               |
| `fp` + 7      | Keyword Argument 0 Value           |               |
| `fp` + 8      | Keyword Argument 1 Keyword         |               |
| `fp` + 9      | Keyword Argument 1 Value           |               |
|  ...          |  ...                               |               |
|               | Argument n - 1                     | `sp` - n      |
|               |  ...                               |  ...          |
|               | Argument 1                         | `sp` - 2      |
|               | Argument 0 (`this`) / Return Value | `sp` - 1      |
|               | < register spill area>             | `sp`          |

Next, the runtime has to identify the specific Method on the specific Class that this message targets, from which we
retrieve:

 * The names, order, and count of expected arguments
 * The pointer to the compiled code to jump into
 * The stack "high water mark," the maximum possible size of the stack (past `sp`) in the callee code

This part of the dispatch process is a high priority target for optimization, as SuperCollider is a message-heavy
programming language. It seems plausible that the compiler could narrow the range of options for some messages
considerably, possibly allowing for dispatch to happen entirely inline in some cases. It feels intuitive that a
transition back into C++ on every message adds unnecessary drag to the system. However, this trades off against
the C++ compiler being able to generate optimized bytecode that is likely much faster in most cases than Hadron
compiled code. Dispatch exceptions will always require a transition to C++ code anyway, so inlining message
dispatch remains a future work item, and all dispatch code routes through C++.

The dispatch code follows the following pseudocode to finalize the stack before method entry:

```
remainingSize <- size remaining in current Stacklet past sp

if remainingSize >= highWaterMark:
    argumentsCopied <- argumentsProvided
    if argumentsProvided < expectedArgumentCount:
        sp <- sp + (expectedArgumentCount - argumentsProvided)
    else if argumentsProvided > expectedArgumentCount:
        argumentsCopied <- expectedArgumentCount
        sp <- sp - (argumentsProvided - expectedArgumentCount)
else:
    allocate new Stacklet
    fp <- first address in new Stacklet
    fp[0] <- caller frame pointer from old Stacklet
    fp[1] <- caller stack pointer from old Stacklet
    fp[2] <- caller return address
    fp[3] <- message selector symbol
    sp <- fp + 4 + expectedArgumentCount
    argumentsCopied <- min(argumentsProvided, expectedArgumentCount)
    copy up to argumentsCopied arguments from old frame into new frame

if argumentsCopied < expectedArgumentCount:
    copy remaining in-order arguments from defaults array to sp-relative argument position

for each key/value pair in keyword arguments:
    lookup key position in method arguments name table
    copy value to sp-relative argument position

if target message has variable arguments:
    create a new Array and copy any remaining input arguments to that
    provide Array value in last argument position
```

This ordering of operations mimics the current argument selection logic in the legacy SuperCollider interpreter. We
first copy default values for any absent in-order arguments, then overwrite any in-order values with their
provided keyword argument values. This algorithm can result in a fair bit of excess copying. For example,
[SinOsc](https://doc.sccode.org/Classes/SinOsc.html) has the `ar` method with signature
`SinOsc.ar(freq: 440.0, phase: 0.0, mul: 1.0, add: 0.0)` which might commonly be called as `SinOsc.ar(freq: 220)`. In
this case, the dispatch code will first copy all four slots of default values and then copy the keyword argument `freq`
to the second argument position, for a total of five slot copies. It may be possible to speed this up significantly with
inline dispatch.

### Stack Overflow

During compilation, Hadron tracks the "high water mark," meaning the highest amount of stack consumption through all
possible pathways of the callee code. Beyond register spills, message dispatch consumes stack space in the callee code
setting up the stack frame for callee message sends. At dispatch time, we need to determine if there's sufficient
space left in the Stacklet for the callee code. If not, we create a new Stacklet and redirect the stack and frame
pointers to the new Stacklet.

### VarArgs

Variable argument messages expect the last argument to be an `Array` containing any additional arguments specified.
The dispatch code creates a new `Array` object from those arguments during stack setup. We then overwrite the final
argument in the stack with a pointer to that new `Array`.

## Primitive Support

## Garbage Collection

# Compilation Stages

## Lexing

## Parsing

## Abstract Syntax Tree Construction

## HIR Generation and the Control Flow Graph

## HIR Optimization Passes

## Block Serialization and Lowering to LIR

## LIR Optimization Passes

## Lifetime Analysis

## Register Allocation

## Allocation Resolution and Move Scheduling

## Machine Code Emission

# Class Library Compilation
