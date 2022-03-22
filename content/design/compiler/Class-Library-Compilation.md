---
title: "Class Library Compilation"
weight: 100
---

The SuperCollider workflow presumes an ahead-of-time class library compilation, and the interpreter automatically
compiles the class library on startup. The SuperCollider development community maintains the official class library in
the [SCClassLibrary directory](https://github.com/supercollider/supercollider/tree/develop/SCClassLibrary) in the
SuperCollider repository. It consists of roughly 350 files and 75k lines of code.

SuperCollider supports object composition through single inheritance. A subclass inherits the class and instance
variables and methods of its superclass. A subclass can override any superclass method and access the
superclass method via the `super` keyword. Therefore, we must first compile all class ancestors before compiling a
derived class.

## Define Class Hierarchy

The compilation process scans the input class files in arbitrary order, so we compile the class library in multiple
passes. The first pass extracts all class metadata from each file and builds Abstract Syntax Trees out of each defined
method. Building the AST allows the class library to unload the file after scanning, freeing that memory for subsequent
passes. We have the class library's complete hierarchy at the end of the first pass.

## Compose Classes From Superclasses

The second pass traverses the class hierarchy from root superclass `Object` to all descendants. This pass defines all
the instance variables by concatenating the subclass instance variables onto a new copy of the superclass variables. By
building the instance variable list in order of superclass to subclass, the compiler can ensure that the indices of
subclass instance variables in inherited superclass methods are still consistent.

Note that class variables and constants in subclasses are *not* composed from superclasses, meaning subclasses don't get
a copy of an inherited class variable but rather can read and modify the superclass class variable itself. Finally, this
pass lowers the AST to Frame objects with HIR and extracts message dependency information from each method.

**A Note About Inlining**

*Dispatch* is the process of mapping an input pair of *target type* and *selector symbol* (message name) to a *class*
and *method*, fixing up the stack to match the input assumptions of the target code and jumping into the entry point of
the method bytecode. SuperCollider heavily relies on message passing for almost all operations, so it follows that
optimizing dispatch would benefit the language speed. Because SuperCollider is a dynamic programming language, the
*target type* is not always known at compile-time, requiring runtime or *dynamic dispatch*. Hadron may narrow the
options with most messages by combining knowledge of the selector and a limited range of target types. And, if there are
only a few options for a message dispatch, Hadron could inline the dispatch decision logic. Hadron could also inline the
complete target method if narrowed to a single target, thus eliminating the dispatch cost.

The class library builds dependency graphs between known callees and methods to support either form of inlining. For
example in the following nonsensical SuperCollider class:

{{< highlight SuperCollider "linenos=table" >}}
Foo {
    bazz {
        var a = Bar.buzz;
        ^a
    }
}
{{< /highlight >}}

The class library creates a dependency from `Foo.bazz` to `Bar.buzz`, ensuring that `Bar.buzz` is compiled first. When
compiling `Foo.bazz`, Hadron can choose to inline the dispatch to `Bar.buzz` or inline the entire `Bar.buzz` method
directly into `Foo.bazz`.

## Compile All Methods

Hadron uses the dependency graph to determine the final compilation order. This final compilation pass takes each method
to JIT bytecode.
