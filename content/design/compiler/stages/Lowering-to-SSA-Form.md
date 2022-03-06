---
title: Translation to SSA Form
weight: 30
---

The `BlockBuilder` class translates the input program from a syntax tree to
[Single Static Assignment](https://en.wikipedia.org/wiki/Static_single_assignment_form) form. SSA form supports analysis of the flow of values
through program execution, including control flow structures like `if` and `while`. SSA form also simplifies
optimization strategies like dead code elimination and constant folding.

Hadron uses two levels of [Intermediate Representation](https://en.wikipedia.org/wiki/Intermediate_representation) for
SSA form code, a high-level form called HIR, for High-level IR, and LIR, for Low-level IR. HIR tracks changes to named
values like local variables, arguments, and member variables, whereas LIR tracks usages of virtual registers further on
in compilation.

## Type Deduction

SuperCollider is a dynamic, message-driven programming language, and the runtime message dispatch system handles routing
messages to the appropriate receiver object. However, if the compiler can derive value *types* at compile-time, this
information can simplify the generated code and unlock powerful optimization techniques. The language has no official
facility for declaring a typed value, but there are ways the compiler can derive types by following the flow of values 
throughout a block.

Consider the following code, taken from the class library file
[Integer.sc](https://github.com/supercollider/supercollider/blob/be060672f394c0a5054075f7318fdc8dedbb57b3/SCClassLibrary/Common/Math/Integer.sc#L235):

{{< highlight plain "lineNos=true,lineNoStart=235" >}}
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

Any character sequence starting with a lower-case alpha character followed by zero or more alphanumeric characters or an
underscore is an *identifier* or *name* describing a variable value. SuperCollider is fairly lenient in allowing
declarations of different variables with identical names. For example, the following code compiles:

{{< highlight plain "lineNos=true" >}}
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

*Note:* For class variables, instance variables, and constants with the same name *within a class*, the compiler always
selects the *first declared* value.

If there is an identical class variable name between a superclass and subclass, the search starts at the subclass and
goes up through the class hierarchy. The net result of the hierarchy search is that overriding class variables hides the
superclass variable. The SuperCollider documentation [discusses
this](https://doc.sccode.org/Guides/WritingClasses.html#Variable%20Scope) too.

For object instance variables, the search happens in the *opposite* direction, from superclass to subclass. So, for
example with these classes defined:

{{< highlight plain "lineNos=true" >}}
M {
    classvar x = 0;
    var a = 0;
}

N : M {
    classvar x = 1;
    var a = 1;

    getX { ^x }
    getA { ^a }
}
{{< /highlight >}}

The interpreter produces the following:

{{< highlight plain "lineNos=true" >}}
(
var n = N.new;
n.getX.postln;  // 1
n.getA.postln;  // 0
)
{{< /highlight >}}

However, the automatically defined accessor methods always resolve to the local variable name, so if we modify `N` in
our example to include a read accessor method on `a`, as follows:

{{< highlight plain "lineNos=true" >}}
M {
    classvar x = 0;
    var a = 0;
}

N : M {
    classvar x = 1;
    var <a = 1;

    getX { ^x }
    getA { ^a }
}
{{< /highlight >}}

The interpreter now produces the following:

{{< highlight plain "lineNos=true" >}}
(
var n = N.new;
n.getX.postln;  // 1
n.getA.postln;  // 0
n.a.postln;     // 1
)
{{< /highlight >}}

### Ephemeral And Persistent Values

CPUs manipulate values in registers, and registers are the fastest form of storage they access, so Hadron always assigns
values to register locations. Local variable lifetimes are limited to their lexical scope, so they exist only in
registers. Member variables are tied to object lifetimes, so we allocate these from the heap. Local variables can be
*captured* by heap-allocated functions. Hadron must identify all persistent values during compilation and guarantee that
they are copied back out to their heap locations on any possible path out of the method.

Hadron accesses all persistent values via heap pointers with offsets known at compile-time:

| Type                              | Initial Value             | Save                                 |
|-----------------------------------|---------------------------|--------------------------------------|
| Local Variable                    | Constant                  | Never                                |
| Arguments                         | Load from stack           | Never                                |
| Captured Local Value (outside)    | Constant                  | Save to new Array                    |
| Captured Local Value (inside)     | Load from context array   | Save to context Array                |
| Instance Vars                     | Load from this pointer    | Save to this pointer                 |
| Class Vars                        | Load from class var table | Save to class var table              |

The Legacy SuperCollider interpreter keeps the entire stack frame from a captured lexical scope, including arguments, and arguments behave exactly like local variables for lexical scoping.

### Imported Names

Once Hadron determines the origin of the named value, it adds the appropriate import HIR statement to the first block
within the control flow graph of the method. Hadron reserves this block for import statements and argument loads. Because Hadron executes this block before any other, it
[dominates](https://en.wikipedia.org/wiki/Dominator_(graph_theory)) all other blocks in the graph, ensuring the
validity and existence of the named value anywhere in the importing frame.

Subsequent uses of the imported name behave exactly like local variables. Hadron determines which values need to be
saved to the heap while lowering the method code to LIR.