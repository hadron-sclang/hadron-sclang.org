---
title: Lowering to SSA Form
weight: 30
---

The `BlockBuilder` class translates the input program from a syntax tree structure to a form supporting type
analysis and optimization. [Single Static Assignment form](https://en.wikipedia.org/wiki/Static_single_assignment_form)
supports analysis of the flow of values and types through program execution, including control flow structures
like `if` and `while`. SSA form also simplifies optimization strategies like dead code eliminiation and constant
folding.
