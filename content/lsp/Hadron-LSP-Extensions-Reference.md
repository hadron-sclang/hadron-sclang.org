---
title: Hadron LSP Extensions Reference
linkTitle: Hadron LSP Extensions Reference
author: Luke Nihlen
type: "posts"
---

# Hadron LSP Extensions Reference

Hadron extends the [Language Server Protocol](https://microsoft.github.io/language-server-protocol) for its own
automation and testing purposes.

## Compilation Diagnostics

method: `hadron/compilationDiagnostics`

params: `HadronCompilationDiagnosticParams` defined as follows:

```typescript
export interface HadronCompilationDiagnosticParams {
    /**
     * Path to the source code file for generation of compilation diagnostic information.
     */
    textDocument: TextDocumentIdentifier;
}
```

response:

```typescript
export interface HadronCompilationDiagnosticsResponse {
    /**
     * The request-id provided with the method call.
     */
    id: integer;

    /**
     * Individual compilation units for the document.
     */
    compilationUnits: array;
}
```

Hadron compiles objects at the Block level, meaning the response can contain more than one compilation unit, such as
class files. The `compilationUnits` member is an array of zero or more `HadronCompilationDiagnosticUnit` objects.

```typescript
export interface HadronCompilationDiagnosticUnit {
    /**
     * The name of the compilation unit. If the input document is an interpreter script
     * this name is "INTERPRET", if a class file, the names are formatted "ClassName:methodName".
    */
    name: string;

    /**
     * The parse tree for this compilation unit.
     */
    parseTree: HadronParseTreeNode;

    /**
     * The Abstract Syntax Tree for this compilation unit.
     */
    ast: HadronASTNode;

    /**
     * The root frame of the control flow graph, including the HIR.
     */
    rootFrame: HadronFrame;

    /**
     * The Linear Frame with LIR, taken through to register allocation.
     */
    linearFrame: HadronLinearFrame;

    /**
     * JIT-compiled generic machine code.
     */
    machineCode: HadronGenericMachineCode;
}
```
