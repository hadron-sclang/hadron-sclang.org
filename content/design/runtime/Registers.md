---
title: "Registers"
---

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