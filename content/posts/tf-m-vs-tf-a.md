---
title: "TF-M vs TF-A — When Do You Reach For Which?"
date: 2026-03-17
draft: false
tags: ["Quick Take", "TF-M", "TF-A", "TrustZone", "Trusted Firmware", "ARM"]
categories: ["Embedded Security"]
description: "TF-M is for Cortex-M microcontrollers, TF-A is for Cortex-A application processors. Different exception models, different attack surfaces, not interchangeable."
summary: "TF-M lives on Cortex-M with SAU and NSC veneers. TF-A lives on Cortex-A with EL3 and BL1/BL2/BL31. Pick by the silicon, not by preference."
ShowToc: false
ShowReadingTime: true
---

> This is personal exploration on publicly available reference platforms; it does not represent the views or work of my employer.

**Short answer:** TF-M is for Cortex-M microcontrollers. TF-A is for Cortex-A application processors. The exception model on your silicon decides which one you run.

People mix these up because both are called "Trusted Firmware" and both ship out of TrustedFirmware.org. They're not interchangeable.

![TF-M vs TF-A architecture comparison](/images/tf-m-vs-tf-a.svg)

## TF-M (Trusted Firmware-M)

Cortex-M class silicon with Armv8-M TrustZone. There are no exception levels — instead the SAU and IDAU split the address space into Secure and Non-Secure regions. TF-M runs as the SPE. The application runs as NSPE and crosses into the SPE through NSC veneers, gated by the SG instruction.

Footprint is tens of KB. Ships with PSA Crypto, Internal Trusted Storage, and Initial Attestation as secure partitions. Lives on sensors, wearables, smart locks, anywhere an MCU runs an RTOS.

## TF-A (Trusted Firmware-A)

Cortex-A class silicon with Armv8-A TrustZone. You get EL0 through EL3 as exception levels, an MMU, and a multi-stage boot chain:

- **BL1** — immutable ROM, anchors the chain of trust
- **BL2** — trusted boot loader, verifies the next stages
- **BL31** — EL3 runtime, handles SMC calls and PSCI
- **BL32** — secure payload, usually OP-TEE
- **BL33** — non-secure bootloader, usually U-Boot or UEFI

Footprint is megabytes. Lives on phones, automotive SoCs, gateways, anywhere Linux runs in the normal world.

## What works

Pick by the silicon. Armv8-M with SAU means TF-M. EL3 means TF-A. If your SoC has both an A-class cluster and an M-class security island — common in automotive and some gateways — you'll run both. They cooperate. They don't replace each other.

## Honest tradeoffs

TF-M is small but isolation is coarser. You're working inside an MPU and SAU, not an MMU with proper privilege levels. TF-A gives you a real secure OS and full privilege separation, but the five-stage boot is not free in either complexity or RAM.

Same family, different worlds.