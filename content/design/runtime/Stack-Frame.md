---
title: "Stack Frame"
---

{{< toc >}}

## Organization

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

Next, the runtime has to identify the method on the class that this message targets, from which we retrieve:

 * The names, order, and count of expected arguments
 * The pointer to the compiled code to jump into
 * The stack "high water mark," the maximum possible size of the stack (past `sp`) in the callee code

The dispatch code follows the following pseudocode to finalize the stack before method entry:

{{< highlight plain "linenos=table" >}}
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
{{< /highlight >}}

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
