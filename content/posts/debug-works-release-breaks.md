---
title: "Why Does My Code Work in Debug But Break in Release?"
date: 2025-09-23
draft: false
tags: ["Quick Take", "Firmware", "Debugging", "Compiler", "Embedded"]
categories: ["Embedded Engineering"]
description: "Short answer: optimization isn't doing the wrong thing. Your code was always wrong — debug builds just happened to hide it."
summary: "The four usual suspects when code works at -O0 and breaks at -O2, in the order to check them."
ShowToc: false
ShowReadingTime: true
---

Short answer: optimization isn't "doing the wrong thing." Your code was always wrong — debug builds just happened to hide it.

The four usual suspects, in the order to check them:

**1. A missing `volatile` on a hardware register or shared variable.**
The compiler decides it can cache the value in a register instead of re-reading it from memory. At `-O0` it didn't bother. At `-O2` it does, and now your ISR's update to that flag is invisible to the main loop.

**2. An uninitialized local that happened to be zero in debug.**
Debug builds often zero stack frames as a side effect. Release builds don't. If your code worked because some `int counter;` was magically zero on entry, it'll stop working the moment that stack slot inherits whatever the previous function left behind.

**3. Undefined behavior the optimizer is now allowed to act on.**
Signed integer overflow, strict aliasing violations, reading past the end of an array — all undefined. At `-O0` the compiler typically generates the literal instructions you wrote. At `-O2` it's free to *assume* the UB doesn't happen and reorder, delete, or "simplify" code accordingly.

**4. Timing that only worked because debug was slow.**
If the bug involves a peripheral, a DMA, or another core, and it appears at `-O2` but not `-O0` — you don't have a compiler bug, you have a race that release just exposed.

What to actually do:

- Turn on `-Wall -Wextra` and read every warning. Suspects 1 and 2 usually warn.
- Build release with `-Og` first instead of jumping straight to `-O2`. Most release-only bugs reproduce at `-Og` and it's still debuggable.
- Run UBSan and ASan if your toolchain supports them. They catch category 3 in seconds.

If after all of that the bug still only reproduces at `-O2`, and not at `-O1` or `-Og`, *then* you've earned the right to suspect the compiler.

Almost always, the compiler's fine.