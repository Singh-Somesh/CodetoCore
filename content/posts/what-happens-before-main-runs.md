---
title: "What Actually Happens Before main() Runs?"
date: 2025-05-03
draft: false
tags: ["Quick Take", "Firmware", "ARM", "Cortex-M", "Startup", "Embedded"]
categories: ["Embedded Engineering"]
description: "A lot. The CPU walks through six setup stages between power-up and your first line of C — and bugs in any of them look very different from each other."
summary: "The six things that happen between reset and main() on a typical Cortex-M, in order, and what goes wrong at each stage."
ShowToc: false
ShowReadingTime: true
---

Short answer: a lot. The CPU walks through six setup stages between power-up and the first line of your `main()` — and bugs in any of them look like everything from "main never runs" to "main runs but the globals are garbage."

Here's the sequence at a glance:

![What happens before main() runs — six startup steps on a Cortex-M](/images/before-main-flowchart.svg)

Now the detail of each step, on a typical ARM Cortex-M part:

**1. The CPU reads the vector table.**
On reset, the core reads the first two words at address 0x00000000 (or wherever VTOR points). Word 0 is the initial stack pointer. Word 1 is the address of the reset handler. The core loads SP, then jumps to the reset handler.

If those two words are wrong — table at the wrong offset, linker placed it somewhere else — the CPU faults before any of your code has run.

**2. `Reset_Handler` runs.**
This is the function the linker placed at the reset vector. On most toolchains it's auto-generated startup code, but you can override it. Everything from here is software.

**3. `SystemInit()` sets up clocks.**
The CPU is running on the default reset clock — usually an internal RC oscillator, often slow. `SystemInit()` switches to your real clock source: external crystal, PLL, whatever the board uses. After this, the CPU is running at full speed.

**4. `.data` is copied from flash to RAM.**
Initialized globals like `int counter = 5;` have their starting value sitting in flash, but they have to live in RAM so they can be modified at runtime. The startup code copies the block from flash to RAM.

**5. `.bss` is zeroed.**
Uninitialized globals like `int counter;` are guaranteed by the C standard to read as zero. The startup code zeroes the entire `.bss` region in RAM to make that true.

**6. Libc init runs, then `main()`.**
On GCC + newlib this is `__libc_init_array` — it calls any constructor functions you've marked, sets up libc internals like the heap, and finally calls `main()`.

Why this matters when things break:

- `main()` never runs → problem in steps 1–3 (vector table, reset handler, or clock setup).
- `main()` runs but globals look wrong → problem in steps 4–5 (`.data` or `.bss`).

Worth setting a breakpoint on `Reset_Handler` once on every new project, just to step through and watch it happen. After that, when something goes weird before `main()`, you'll know exactly where to look.