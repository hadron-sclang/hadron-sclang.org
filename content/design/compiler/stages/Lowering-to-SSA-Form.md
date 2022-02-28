---
title: Lowering to SSA Form
weight: 30
---

The `BlockBuilder` object is responsible for combining the input Abstract Syntax Tree with other information about the
class library and lowering the code to Single Static Assignment form. The output is HIR in a CFG.