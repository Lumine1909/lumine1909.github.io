---
layout: post
title: "How does Graal make the optimization in Reflexion possible"
date: 2000-01-03 00:00:00 +0000
categories: [General]
---

My [previous article](../../posts/How-does-Reflexion-make-non-SF-case-faster/) only explained how I implement the optimization, and this article is write to explain how GraalVM make it possible.

To verify that Graal actually performs these optimizations, we can inspect the generated machine code. Luckily, [there is a way to read the assembly generate by Graal JVMCI JIT](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html), so we can check what is the code becomes after JVMCI compiled it.

By setting up a directive file like this
```json
[
  {
    match: "*::*Call*",
    PrintAssembly: true,
    PrintInlining: true
  }
]
```
And run the benchmark with `-XX:+UnlockDiagnosticVMOptions -XX:PrintAssemblyOptions=intel -XX:CompilerDirectivesFile=directives.json` VM options, we can check the [generated assembly code](../../assets/raw/output_jvmci.txt).

WIP...
