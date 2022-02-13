---
title: Hadron Design Document
linkTitle: Hadron Design Document
author: Luke Nihlen
type: "posts"
---

## Slots

## Runtime Environment

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
only for Hadron code. Hadron reserves 3 GPRs, `GPR0`, `GPR1` and `GPR2`, as pointers to the `ThreadContext` structure
and the Hadron frame and stack pointers. Outside of these, all registers are *caller-save*, meaning the callee can
change any of their values at will.

Hadron can transition from compiled C++ code into compiled SuperCollider bytecode and back out again via a *trampoline*,
code responsible for maintaining the state of the C++ program stack and all registers and then jumping to the Hadron
bytecode. Because the transition between stacks requires saving and restoring of state it is relatively expensive, so we
take care to minimize them to only those that are necessary.

### Stack Frame

Hadron allocates large-size `Frame` objects and adds them to the root set for scanning during garbage collection.
Excepting register spills, Hadron saves all values on the stack in `Slot` format, meaning they are all 8 bytes
with the header type tags included.

Stack addresses grow upward, starting with the value in the frame pointer and ending with the register spill area past
the stack pointer. Hadron combines multiple stack frames in each `Frame` object, with the hopes that most times
during message dispatch the runtime doesn't have to trampoline back to C++ code to allocate additional memory as the
stack grows.

On method entry the stack is laid out as follows:

| frame pointer | contents               | stack pointer |
|---------------|------------------------|---------------|
| `fp`          | Caller Frame Pointer   |               |
| `fp` + 1      | Caller Stack Pointer   |               |
| `fp` + 2      | Caller Return Address  |               |
|    ...        | < dispatch work area > |  ...          |
|               | Argument 0 (this)      | `sp` - n      |
|               |  ...                   |  ...          |
|               | Argument n             | `sp` - 1      |
|               | < register spill area> | `sp`          |

The dispatch work area can be 3 or more slots in size, so that variable size is accounted for by maintaining two
separate pointers into the stack. The frame pointer indicates the beginning of the stack frame, and the stack pointer
indicates the division between the in-order argument values and the register spill area, a temporary area of memory used
to save register values during method calls and register allocation overflow.

Method calls always include a `this` pointer provided as the first argument. To save a copy, and s `this` is also the
default return value of all messages, the argument 0 slot is also where this message stores the return value if it is
different from `this`.

### Function Dispatch

### Primitive Support

## Garbage Collection

