---
title: "Is SysTick Accurate Enough for Microsecond Timing?"
date: 2026-03-12
draft: false
tags: ["Quick Take", "Cortex-M", "SysTick", "Timing", "Firmware"]
categories: ["Embedded Engineering"]
description: "Short answer: the counter is. Everything around it usually isn't."
summary: "What SysTick gets right at microsecond resolution, what bites you, and when to reach for a hardware timer instead."
ShowToc: false
ShowReadingTime: true
---

Short answer: the *counter* is. Everything around it usually isn't.

SysTick on a Cortex-M is a 24-bit downcounter clocked off the core. At a 64 MHz core clock, one tick is ~15.6 ns. So the resolution is plenty — at microsecond scale, the count itself is fine.

What actually limits you:

**Interrupt latency.**
If you're using SysTick to fire an interrupt every microsecond, the time between the counter hitting zero and your handler's first useful instruction is dozens of cycles — context save, vector fetch, pipeline refill. That's typically 100–200 ns of overhead per tick on most parts, and it's not constant. Higher-priority ISRs make it worse.

**Preemption by other ISRs.**
SysTick is usually configured at a low priority. Any ISR running at the same priority or higher delays your tick. UART, ADC, and timer interrupts all in flight at once means "every microsecond" quietly becomes "every microsecond plus whatever ran first."

**Clock domains.**
SysTick counts core ticks. Your peripheral lives on PCLK or some other bus clock. If you're trying to time a peripheral event with SysTick precision, remember that the peripheral's view of "now" is on a different, usually slower, clock.

What works:

- For *software-to-software* intervals: SysTick polling (not interrupting) is fine down to a few microseconds.
- For *peripheral events*: use a hardware timer's input capture, not SysTick. The capture happens in hardware on the same edge the peripheral sees.
- For *one-shot delays*: a busy-loop on `SysTick->VAL` is more accurate than an interrupt-driven one, because there's no ISR overhead in the path.

The honest answer: SysTick is fine for microsecond timing if you're measuring software intervals and can tolerate jitter of a few hundred nanoseconds. For anything tighter, or anything involving an external signal, reach for a hardware timer.

*This is a general engineering note on publicly available platforms; it does not represent the views or work of my employer.*