---
title: Translation to SSA Form
weight: 30
---

The `BlockBuilder` class translates the input program from a syntax tree to [Single Static Assignment
form](https://en.wikipedia.org/wiki/Static_single_assignment_form). SSA form supports analysis of the flow of values through program execution, including control flow structures like `if` and `while`. SSA form also simplifies
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
or more alphanumeric characters or an underscore as an *identifier* or *name* describing a variable value. Identifiers
describe variables in a variety of contexts, in order of precedence from highest to lowest:

* local variables declared within a method with the `var` keyword
* arguments provided to methods with the `arg` keyword or pipe `|` symbol
* instance variables declared in classes with the `var` keyword
* class variables declared in classes with the `classvar` keyword

SuperCollider is fairly lenient in allowing declarations of different variables with identical names. For example, the
following code compiles:

{{< highlight plain "lineos=table" >}}
XX {
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


|                   | Class Vars | Instance Vars | Arguments | Local Vars |
|-------------------|------------|---------------|-----------|------------|
| **Class Vars**    | ambiguous  | instance      | argument  | local      |
| **Instance Vars** | instance   | ambiguous     | argument  | local      |
| **Arguments**     | argument   | argument      | **error** | **error**  |
| **Local Vars**    | local      | local         | **error** | **error**  |