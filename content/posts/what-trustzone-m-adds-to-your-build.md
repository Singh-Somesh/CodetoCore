---
title: "What TrustZone-M Adds to Your Build"
date: 2025-12-03
draft: false
tags: ["TrustZone-M", "Cortex-M", "Cortex-M23", "Cortex-M33", "ARM", "Embedded Security", "Secure Boot"]
categories: ["Embedded Security"]
description: "Why moving from a plain Cortex-M to a TrustZone-M part doubles your project structure — and what each new piece is actually doing."
summary: "A walk through what TrustZone-M adds to a build: two projects, two memory maps, SAU configuration, the veneer region, and signed images. None of it is gratuitous."
ShowToc: true
ShowReadingTime: true
---

If you've shipped firmware on a Cortex-M0/M3/M4 part, you know what to expect from the build: one project, one linker script, one binary. Flash it, done.

Move to a Cortex-M23 or M33 with TrustZone-M and the build roughly doubles. Two projects. Two linker scripts. Two binaries. A pile of new configuration you didn't have before. None of it is gratuitous — TrustZone-M is doing something real — but it isn't always obvious *why* each piece exists.

Here's what's actually being added, and why.

## Two Projects, Not One

The core thing: TrustZone-M splits the MCU into a Secure world and a Non-Secure world, enforced by hardware. They can't share code or data freely — that's the whole point. So the firmware comes from two separate compile units:

- A **secure project** — trusted code, secret keys, security-critical state.
- A **non-secure project** — your application, your RTOS, your networking, your bugs.

Each builds independently into its own binary. They run on the same processor, sit in the same flash, but the hardware enforces that one can't read the other's memory.

This is the irreducible reason the build is bigger. Everything else falls out of it.

## Two Memory Maps

One MCU, but flash and RAM both get split. The layout looks roughly like this:

```
FLASH

  ┌────────────────────┐  ← high address
  │                    │
  │  Non-Secure Code   │   NS
  │  (your app)        │
  │                    │
  ├────────────────────┤
  │  NSC Veneers       │   NSC
  ├────────────────────┤
  │                    │
  │  Secure Code       │   S
  │  (reset vector)    │
  │                    │
  └────────────────────┘  ← low address


RAM

  ┌────────────────────┐  ← high address
  │                    │
  │  Non-Secure Data   │   NS
  │  (app heap/stack)  │
  │                    │
  ├────────────────────┤
  │                    │
  │  Secure Data       │   S
  │  (keys, state)     │
  │                    │
  └────────────────────┘  ← low address
```

The CPU comes out of reset in secure state, jumps to the secure reset vector at the bottom of flash, configures the SAU, and only then hands control to the non-secure side. The NSC region sits between the two code regions — it's the only place the non-secure side is allowed to enter secure code from, via the `SG` (Secure Gateway) instruction.

Each project's linker script only knows about its own regions. Get the addresses wrong — overlap, or a gap that crosses a security boundary — and you don't get a friendly error. You get a hard fault at runtime, usually somewhere unhelpful.

## The SAU Has to Be Programmed

A regular Cortex-M doesn't have a Security Attribution Unit. A TrustZone-M part does, and it has to be told which addresses are secure, which are non-secure, and which are Non-Secure-Callable.

This happens very early in the secure firmware's boot path. The secure side owns this configuration — the non-secure side has no say. And it has to match the memory map exactly. If the SAU thinks an address is secure but the linker placed non-secure code there, the CPU faults the moment that code tries to run.

So now you have three things that all have to stay in sync, by hand, across two projects:

- The memory map.
- A linker script for each project that follows it.
- SAU configuration in the secure project that also follows it.

> Treat the memory map as the source of truth and have the rest derive from it. Duplicating addresses across files is how you find out at 2 a.m. that one of them was off by 4 KB.

## Veneers Are Real Code in the Binary

When the non-secure side calls a function in the secure side, it doesn't jump straight in. It jumps to a **veneer** — a tiny stub in the NSC region that executes the `SG` instruction to legally cross the boundary.

The veneers live in NSC memory. The non-secure side has to know their addresses to link against them. The secure side has to declare which functions are exported as veneers and which aren't.

In practice:

- The secure project produces an import library or header describing the veneers.
- The non-secure project links against that.
- Every time you add or remove a secure entry point, both sides have to rebuild in lockstep.

Forgetting to rebuild one side after changing the interface is a classic source of "it builds fine but hard-faults on boot."

## Secure Boot Adds Signing

If the product enforces secure boot — and on a TrustZone-M part that's shipping, it usually should — the build also has to *sign* the images. Now you have opinions to make about:

- Which signing key.
- Where it lives — dev keys in the repo are fine, production keys belong in an HSM. They should not be the same key.
- The image format the bootloader expects: header, signature, version metadata.

That often brings a third binary into the picture: the bootloader itself, which verifies the signed images at boot. Another linker script, another memory region, another set of keys to manage.

## Honest Tradeoffs

None of this is overhead for its own sake:

- Two projects, because the hardware enforces the split.
- Two memory maps and SAU config, because that's how the hardware *knows* what's secure.
- Veneers, because a direct call across the boundary would defeat the isolation.
- Signing, because unsigned firmware in a secure-boot system is just expensive paperweights.

What you're paying for is a property that's hard to get any other way: a fully compromised application can't read your secure keys, your secure storage, or your attestation material. If you don't need that property, you don't need TrustZone-M. If you do, the extra build complexity is the price.

## What Works

A few habits that pay off once you've done this a couple of times:

- Keep the memory map in one place that both projects pull from. Don't duplicate addresses across linker scripts by hand.
- Rebuild both sides whenever the secure interface changes. CI is the right place to enforce this.
- Treat signing-key configuration as a security review item, not a build engineering one.
- Pin everything — toolchain version, bootloader version, the lot.

It doesn't make the build small. It just makes it predictable.

---

*This is personal exploration on publicly available reference platforms; it does not represent the views or work of my employer.*