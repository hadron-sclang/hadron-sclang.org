---
title: Translation to SSA Form
weight: 30
---

The `BlockBuilder` class translates the input program from a syntax tree to [Single Static Assignment]
(https://en.wikipedia.org/wiki/Static_single_assignment_form) form. SSA form supports analysis of the flow of values through program execution, including control flow structures like `if` and `while`. SSA form also simplifies
optimization strategies like dead code elimination and constant folding.

Hadron uses two levels of [Intermediate Representation](https://en.wikipedia.org/wiki/Intermediate_representation) for
SSA form code, a high-level form called HIR, for High-level IR, and LIR, for Low-level IR. HIR tracks changes to named
values like local variables, arguments, and member variables, whereas LIR tracks usages of virtual registers further on
in compilation.

## Type Deduction

SuperCollider is a dynamic, message-driven programming language, meaning that no types are specified at compile time,
and the runtime dispatch system handles routing messages to the appropriate receiver object. However, inlining and
optimizing code needs to understand the possible types of any values that it operates on. Furthermore, the types of some
values are known at compile time, particularly literals.

Consider the following code, taken from the class library file `Integer.sc`:

{{< highlight plain "linenos=table" >}}
factors {
    var num, array, prime;
    if(this <= 1) { ^[] }; // no prime factors exist below the first prime
    num = this.abs;
    // there are 6542 16 bit primes from 2 to 65521
    6542.do {|i|
        prime = i.nthPrime;
        while { (num mod: prime) == 0 }{
            array = array.add(prime);
            num = num div: prime;
            if (num == 1) {^array}
        };
        if (prime.squared > num) {
            array = array.add(num);
            ^array
        };
    };
    // because Integer is 32 bit, and we have tested all 16 bit primes,
    // any remaining number must be a prime.
    array = array.add(num);
    ^array
}
{{< /highlight >}}

## Locating Named Values

The SuperCollider grammar treats any character sequence starting with a lower-case alpha character and followed by zero
or more alphanumeric characters or an underscore as an *identifier* or *name* describing a variable value. SuperCollider is fairly lenient in allowing declarations of different variables with identical names. For example, the
following code compiles:

{{< highlight plain "lineos=table" >}}
XX {
    const x = -1;
    const x = 0;

    classvar x = 1;
    classvar x = 2;

    var x = 3;
    var x = 4;

    func {
        var x = 5;
        var f = {
            var x = 6;
            var g = {
                var x = 7;
                x.postln;
            };
            g.value();
        };
        f.value();
    }
}
{{< /highlight >}}

To resolve `x` in the `x.postln` call, the interpreter will look first in locally-scoped variables, so in this case a
call to `func` will always print `7`. The identifier matching algorithm searches in order:

* local variables declared within a method with the `var` keyword, from innermost scope outward to root scope
* arguments provided to methods with the `arg` keyword or pipe `|` symbol
* instance variables declared in classes with the `var` keyword
* class variables declared in classes with the `classvar` keyword
* constants declared in classes with the `const` keyword

Put another way, for any two identifiers with duplicate names, say `x` in our code example, the algorithm will select
the following:

|                   | Local Vars | Arguments | Instance Vars | Class Vars | Constants |
|-------------------|------------|-----------|---------------|------------|-----------|
| **Local Vars**    | **error**  | **error** | local         | local      | local     |
| **Arguments**     | **error**  | **error** | argument      | argument   | argument  |
| **Instance Vars** | local      | argument  | *first*       | instance   | instance  |
| **Class Vars**    | local      | argument  | instance      | *first*    | class     |
| **Constants**     | local      | argument  | instance      | class      | *first*   |

*Note:* For class variables, instance variables and constants with the same name, the value selected is always the
*first declared*. This includes for derived classes that re-declare a member variable with the same name as an inherited
variable; the inherited variable is first declared, and so will always be selected over the derived variable of the same
name.

### Ephemeral And Persistent Values

CPUs are largely constrained to manipulate values in registers, and those registers also are the fastest form of storage
on any modern computer. As a result Hadron always assigns values to register locations. This works well for most local
variables, which are not retained after method return, but member variables and local variables with lexical scoping
have to persist beyond the method. The stack frame is also ephemeral, so all persistent vales must be allocated from the
heap.

Furthermore, Hadron must identify all persistent values during compilation, and guarantee that all register-bound values
are copied back out to their heap locations on all possible paths out of the method. Lexical scoping also means that
local variables which would normally be considered ephemeral can also be *captured*, which Hadron also needs to detect
and add them to the list of persisted values, likely in a new Array that can be attached as context to the capturing
FunctionDef.

It's useful to observe that all of the persistent variables can be accessed as static offsets from pointers - class
variables are an offset against the global class variable table, instance variables are an offset against the instance
`this` pointer, arguments are relative to the stack, and captured local variables are against a newly-created `Array`
instance provided as the context for the `Function`.

| Type                              | Initial Value             | Save                                 |
|-----------------------------------|---------------------------|--------------------------------------|
| Local Variable                    | Constant                  | Never                                |
| Captured Local Variable (outside) | Constant                  | Save to new Array at time of capture |
| Captured Local Variable (inside)  | Load from context array   | Save to context Array                |
| Arguments                         | Load from stack           | *If captured*                        |
| Instance Vars                     | Load from this pointer    | Save to this pointer                 |
| Class Vars                        | Load from class var table | Save to class var table              |

The Legacy SuperCollider interpreter keeps the entire stack frame from a captured lexical scope. This includes
arguments, so it follows that for the purposes of capture arguments behave exactly like local variables; it is possible
to modify argument values inside the capturing function, and they will persist between calls just like captured local
variables.

Decision: at block building time, we load all arguments and then treat them like local variables. So keep
LoadArgumentHIR. Unless captured, these are all *ephemeral* values.
For instance and class variables along with potentially captured external local values we add a LoadExternalHIR or
something to indicate that we have a dependency on those things. Add a bit to the NamedValue to indicate if a value is
ephemeral or not. If not, could also supply a pointer back to the BindExternalHIR..

Or add a new struct that has name and origin enum, these are ExternalValues. They are kept at Frame level. Then
NamedValue inside of HIR can keep a pointer back to them, and if that pointer is non-null it indicates the connection
back to that value and a persistent value status. Then LoadExternalHIR represents a loading of those external values,
and StoreExternalHIR a save.

Frames can be *inlined*, in which case the loads/stores for external local variables become aliases/no-ops.